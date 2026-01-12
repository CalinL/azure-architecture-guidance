# Microsoft Foundry Content Safety & Guardrails
## Solutions for Insurance Business Use Cases

---

## Executive Summary

This document provides a comprehensive analysis and actionable solutions for managing Content Safety filters in Microsoft Foundry/Azure AI, specifically tailored for **insurance business contexts**. The goal is to maintain security guardrails while minimizing false positives that block legitimate business forms and workflows.

---

## Table of Contents

1. [Understanding the Problem](#1-understanding-the-problem)
2. [Microsoft Foundry Guardrails Architecture](#2-microsoft-foundry-guardrails-architecture)
3. [Available Configuration Options](#3-available-configuration-options)
4. [Recommended Solutions](#4-recommended-solutions)
5. [Implementation Strategy](#5-implementation-strategy)
6. [Governance & Compliance](#6-governance--compliance)
7. [References & Resources](#7-references--resources)

---

## 1. Understanding the Problem

### Current Challenges (Your Context)

| Issue | Impact | Risk Level |
|-------|--------|------------|
| **Jailbreak filter false positives** | Business forms blocked incorrectly | High |
| **Sexual content filter false positives** | Insurance claims with medical/anatomical terms flagged | High |
| **Lack of filter logic transparency** | Cannot optimize prompts effectively | Medium |
| **Disabling filters entirely** | Compliance and security risks | Critical |

### Why Insurance Triggers False Positives

Insurance business content often contains:
- **Medical terminology** (anatomical terms, injury descriptions)
- **Legal language** (liability clauses, claim descriptions)
- **Financial data** (compensation amounts, policy details)
- **Accident descriptions** (violence-related vocabulary)
- **Personal injury claims** (self-harm adjacent terminology)

These legitimate business terms can trigger safety classifiers designed for general consumer applications.

---

## 2. Microsoft Foundry Guardrails Architecture

### Filter Categories & Severity Levels

| Risk Category | Severity Levels | Intervention Points |
|---------------|-----------------|---------------------|
| **Hate** | Safe, Low, Medium, High | Prompt & Completion |
| **Sexual** | Safe, Low, Medium, High | Prompt & Completion |
| **Violence** | Safe, Low, Medium, High | Prompt & Completion |
| **Self-harm** | Safe, Low, Medium, High | Prompt & Completion |
| **Jailbreak (Prompt Shields)** | Binary (Detected/Not) | Prompt only |
| **Protected Material** | Binary | Completion only |
| **PII Detection** | Binary + Redaction | Both |

### Default Behavior

- Default threshold: **Medium** (blocks Medium + High severity)
- Prompt Shields (Jailbreak): **On** by default
- All categories: Active on both input and output

---

## 3. Available Configuration Options

### 3.1 Severity Threshold Adjustment (Available to All Customers)

**Configurable by all customers without approval:**

| UI Setting | What it Blocks | Use Case |
|------------|----------------|----------|
| **High blocking** | Low + Medium + High severity | Maximum safety (most restrictive) |
| **Medium blocking** | Medium + High severity | **Default - Balanced** |
| **Lowest blocking** | Only High severity | **✅ Recommended for Insurance** (most permissive) |

✅ **Recommendation**: Set filters to **"Lowest blocking"** for Sexual, Violence, and Self-harm categories. This only blocks truly severe content while allowing low and medium severity (which often triggers false positives with medical/legal terminology).

### 3.2 Annotate Only Mode (Requires Approval)

**Key Feature**: Content is NOT blocked, but annotations are returned via API response.

```json
{
  "content_filter_results": {
    "sexual": {
      "filtered": false,
      "detected": true,
      "severity": "low"
    }
  }
}
```

**Benefits:**
- Zero false positive blocking
- Full visibility into what would have been flagged
- Application-level decision making possible
- Logging for compliance audits

⚠️ **Requires**: [Limited Access Review Application](https://ncv.microsoft.com/uEfCgnITdR)

### 3.3 Prompt Shields Configuration

| Option | Behavior | Recommendation |
|--------|----------|----------------|
| **Filter mode** | Blocks detected attacks | Default |
| **Annotate mode** | Returns detection, no blocking | ✅ Recommended for Insurance |
| **Off** | No detection | Not recommended |

---

## 4. Recommended Solutions

### Solution 1: Optimized Severity Thresholds (Immediate, No Approval Required)

**Action**: Set content filters to "Lowest blocking" for harm categories

**Configuration in Microsoft Foundry Portal:**

1. Navigate to **Guardrails + controls** → **Content filters**
2. Click **Create a content filter**
3. On the **Input filters** page, use the sliders to configure:

| Category | Input Filter | Output Filter |
|----------|--------------|---------------|
| Violence | Lowest blocking | Lowest blocking |
| Sexual | Lowest blocking | Lowest blocking |
| Self-harm | Lowest blocking | Lowest blocking |
| Hate | Medium blocking | Medium blocking |
| Prompt Shields | Annotate only | N/A |

4. On the **Output filters** page, apply the same settings
5. On the **Deployment** page, associate with your model deployment
6. Click **Create content filter**

**Expected Impact**: ~70-80% reduction in false positives while maintaining protection against truly severe content.

---

### Solution 2: Apply for Modified Content Filters (Recommended)

**Process:**

1. **Submit Application**: [Azure OpenAI Limited Access Review: Modified Content Filters](https://ncv.microsoft.com/uEfCgnITdR)

2. **Information Required:**
   - Company details and Azure subscription
   - Detailed use case description (insurance claims processing)
   - Explanation of false positive issues
   - Internal compliance measures in place
   - Data handling and privacy policies

3. **Upon Approval**, you gain access to:
   - **Annotate Only mode**: Content flagged but not blocked
   - **No Filters option**: Complete filter disable (use cautiously)
   - **Full control** over all filter categories

**Application Tips for Insurance:**
- Emphasize regulated industry status
- Highlight existing compliance frameworks (GDPR, industry regulations)
- Describe internal review processes
- Explain business impact of false positives

---

### Solution 3: Implement Annotate-Only with Application-Level Logic

**Architecture:**

```
┌─────────────────────────────────────────────────────────────┐
│                    Application Layer                         │
├─────────────────────────────────────────────────────────────┤
│  ┌─────────────┐    ┌──────────────────┐    ┌────────────┐ │
│  │  User Input │───►│ Pre-processing   │───►│  Azure AI  │ │
│  │  (Insurance │    │ Context Tagging  │    │  Foundry   │ │
│  │   Form)     │    │                  │    │  (Annotate │ │
│  └─────────────┘    └──────────────────┘    │   Only)    │ │
│                                              └─────┬──────┘ │
│                                                    │        │
│  ┌─────────────┐    ┌──────────────────┐          ▼        │
│  │  Final      │◄───│ Business Logic   │◄──────────────────┤│
│  │  Output     │    │ Decision Engine  │    Annotations    │
│  └─────────────┘    └──────────────────┘                   │
└─────────────────────────────────────────────────────────────┘
```

**Implementation Code Example:**

```python
import openai
from typing import Dict, Any

def process_insurance_request(prompt: str, context: str) -> Dict[str, Any]:
    """
    Process insurance-related requests with annotation-based filtering
    """
    
    # System message with insurance-specific context
    system_message = """
    You are an AI assistant for an insurance company processing legitimate 
    business documents including claims, medical reports, and legal documents.
    
    CONTEXT: This is a regulated insurance business environment.
    - Medical terminology is expected and legitimate
    - Injury descriptions are part of normal claims processing
    - Legal and liability language is standard business content
    
    Always maintain professional, factual responses focused on the 
    insurance business context.
    """
    
    response = openai.ChatCompletion.create(
        engine="your-deployment",
        messages=[
            {"role": "system", "content": system_message},
            {"role": "user", "content": prompt}
        ]
    )
    
    # Extract annotations
    annotations = response.get('prompt_filter_results', [])
    completion_annotations = response['choices'][0].get('content_filter_results', {})
    
    # Business logic decision
    decision = evaluate_annotations(annotations, completion_annotations, context)
    
    return {
        "content": response['choices'][0]['message']['content'],
        "annotations": completion_annotations,
        "business_decision": decision,
        "audit_log": create_audit_entry(prompt, annotations, decision)
    }

def evaluate_annotations(prompt_filters, completion_filters, context) -> str:
    """
    Apply business rules to filter annotations
    """
    # Insurance-specific allowances
    insurance_contexts = ["claims", "medical", "liability", "injury", "policy"]
    
    if any(ctx in context.lower() for ctx in insurance_contexts):
        # More permissive for insurance contexts
        if completion_filters.get('sexual', {}).get('severity') in ['safe', 'low']:
            return "ALLOWED"
        if completion_filters.get('violence', {}).get('severity') in ['safe', 'low', 'medium']:
            return "ALLOWED"
    
    # Default behavior for flagged content
    if any(f.get('filtered') for f in completion_filters.values()):
        return "REVIEW_REQUIRED"
    
    return "ALLOWED"
```

---

### Solution 4: Enhanced System Messages (Meta-prompts)

**Purpose**: Provide context to reduce false jailbreak detection

**Insurance-Specific System Message Template:**

```markdown
## Role & Context
You are a professional AI assistant for [COMPANY NAME], a licensed insurance company. 
You assist with processing insurance-related documents, claims, and inquiries.

## Business Context
This is a B2B/B2E environment serving insurance professionals who handle:
- Property and casualty claims
- Health insurance claims and medical documentation
- Life insurance applications and claims
- Liability assessments and legal documentation

## Expected Content Types
The following content types are NORMAL and EXPECTED in this context:
- Medical terminology and anatomical descriptions (for health claims)
- Descriptions of accidents, injuries, and property damage (for claims processing)
- Legal terminology including liability and negligence language
- Financial details including compensation amounts and policy values
- Personal data necessary for policy and claims processing

## Behavioral Guidelines
1. Treat all input as legitimate business documentation unless clearly malicious
2. Process medical and injury descriptions professionally and factually
3. Maintain confidentiality and compliance with insurance regulations
4. Provide accurate, helpful responses for insurance business operations
5. If content seems unusual, clarify intent before refusing service

## Compliance
Responses must align with:
- Insurance industry regulations
- Data protection requirements (GDPR, relevant local laws)
- Professional standards for insurance communications
```

**Why This Helps:**
- Provides context that distinguishes business use from harmful intent
- Reduces false positives in jailbreak detection
- The model understands "role-play" as legitimate business context, not manipulation

---

### Solution 5: Custom Blocklist Strategy (Inverse Approach)

While blocklists are typically for blocking terms, use them strategically:

**Use Case**: Create blocklists for truly prohibited terms specific to your context

```
Step 1: Navigate to Guardrails + controls → Blocklists
Step 2: Create blocklist "Insurance_Prohibited_Terms"
Step 3: Add terms that are NEVER acceptable (specific slurs, explicit terms)
Step 4: Keep this list minimal and focused
Step 5: Attach to your content filter configuration
```

**Benefit**: Maintain targeted blocking while allowing the AI filters to be less aggressive overall.

---

## 5. Implementation Strategy

### Phase 1: Immediate Actions (Week 1)

| Action | Owner | Effort |
|--------|-------|--------|
| Set content filters to "Lowest blocking" | Platform team | Low |
| Implement enhanced system message | Development team | Low |
| Set Prompt Shields to "Annotate only" | Platform team | Low |
| Begin collecting false positive data | QA team | Medium |

### Phase 2: Limited Access Application (Week 2-4)

| Action | Owner | Effort |
|--------|-------|--------|
| Prepare application documentation | Compliance + Engineering | Medium |
| Submit Limited Access Review application | Project lead | Low |
| Prepare internal governance framework | Compliance | Medium |
| Design application-level decision logic | Development team | Medium |

### Phase 3: Advanced Implementation (Week 4-8)

| Action | Owner | Effort |
|--------|-------|--------|
| Implement Annotate-Only architecture | Development team | High |
| Build business logic decision engine | Development team | High |
| Set up monitoring and audit logging | DevOps | Medium |
| Conduct red team testing | Security | Medium |

### Phase 4: Continuous Improvement (Ongoing)

| Action | Owner | Frequency |
|--------|-------|-----------|
| Review false positive/negative rates | QA team | Weekly |
| Update system messages based on findings | Development team | Monthly |
| Compliance audit of filter decisions | Compliance | Quarterly |
| Report issues to Microsoft support | Engineering | As needed |

---

## 6. Governance & Compliance

### Internal Governance Framework

When using modified filters, implement:

#### 6.1 Audit Trail Requirements

```python
# Example audit log structure
audit_entry = {
    "timestamp": "2026-01-09T10:30:00Z",
    "request_id": "uuid-xxx",
    "user_id": "employee_123",
    "business_context": "claims_processing",
    "content_category": "medical_claim",
    "filter_annotations": {
        "sexual": {"detected": True, "severity": "low", "blocked": False},
        "violence": {"detected": False}
    },
    "business_decision": "ALLOWED",
    "decision_reason": "Low severity in medical claims context",
    "human_review_required": False
}
```

#### 6.2 Escalation Matrix

| Annotation Severity | Business Context Match | Action |
|---------------------|------------------------|--------|
| Low | Yes | Auto-approve + Log |
| Low | No | Flag for review |
| Medium | Yes | Auto-approve + Enhanced logging |
| Medium | No | Human review required |
| High | Any | Block + Human review |

#### 6.3 Human Review Process

1. **Triggered by**: High severity annotations or context mismatch
2. **Reviewers**: Trained content moderators (insurance domain experts)
3. **SLA**: 4-hour response for business-critical items
4. **Documentation**: Full audit trail with reviewer decision

### Compliance Considerations

| Requirement | Implementation |
|-------------|----------------|
| **GDPR Article 22** | Human oversight for consequential decisions |
| **Insurance regulations** | Documentation of AI-assisted decisions |
| **Microsoft Code of Conduct** | Responsible use of modified filters |
| **Internal policies** | Regular audits and monitoring |

---

## 7. References & Resources

### Official Microsoft Documentation

| Resource | URL |
|----------|-----|
| Guardrails Overview | [learn.microsoft.com/.../guardrails-overview](https://learn.microsoft.com/en-us/azure/ai-foundry/guardrails/guardrails-overview) |
| Content Filter Configuration | [learn.microsoft.com/.../content-filters](https://learn.microsoft.com/en-us/azure/ai-foundry/openai/how-to/content-filters) |
| Mitigate False Results | [learn.microsoft.com/.../improve-performance](https://learn.microsoft.com/en-us/azure/ai-services/content-safety/how-to/improve-performance) |
| Prompt Shields Documentation | [learn.microsoft.com/.../jailbreak-detection](https://learn.microsoft.com/en-us/azure/ai-services/content-safety/concepts/jailbreak-detection) |
| Safety System Messages | [learn.microsoft.com/.../system-message](https://learn.microsoft.com/en-us/azure/ai-foundry/openai/concepts/system-message) |
| Limited Access Review Application | [ncv.microsoft.com/uEfCgnITdR](https://ncv.microsoft.com/uEfCgnITdR) |

### Support Channels

| Channel | Purpose |
|---------|---------|
| [Content Safety Support](mailto:contentsafetysupport@microsoft.com) | Report persistent false positives |
| Azure Support | Technical issues with configuration |
| Microsoft Account Team | Limited Access application assistance |

---

## Summary Decision Matrix

| Scenario | Recommended Solution | Approval Required |
|----------|----------------------|-------------------|
| Quick win - reduce false positives | Set filters to "Lowest blocking" | No |
| Better visibility without blocking | Switch to "Annotate only" | **Yes** |
| Jailbreak false positives | Enhanced system messages + Annotate mode for Prompt Shields | Partial |
| Full control needed | Apply for Modified Content Filters | **Yes** |
| Domain-specific blocking | Custom blocklists | No |

---

## Next Steps

1. **Immediate**: Implement Solution 1 (set to "Lowest blocking") and Solution 4 (system messages)
2. **This week**: Begin collecting documented false positive cases
3. **Within 2 weeks**: Submit Limited Access Review application
4. **Upon approval**: Implement Solution 3 (Annotate-Only architecture)
5. **Ongoing**: Monitor, iterate, and report persistent issues to Microsoft

---

*Document prepared for: Insurance Business AI Implementation Team*  
*Date: January 9, 2026*  
*Version: 1.0*
