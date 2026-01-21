
# Boundless Books – Performance‑Critical Architecture Decision (AWS)
*A proposal‑ready architecture decision slice with diagrams.*

---

## 1. Performance‑Critical Problem

Boundless Books must deliver **global, low‑latency digital content** (e‑books, interactive media, DRM‑protected assets) to B2C and B2B channels while legacy systems remain operational.

### Performance Constraints
- **p95 ≤ 200 ms** globally  
- **10× burst scaling** within 5 minutes  
- **99.95% availability**  
- Cost optimization required for high‑egress content

---

## 2. Key Non‑Functional Requirements (NFRs)

### NFR Overview Diagram
```mermaid
mindmap
  root((NFRs))
    Performance
      "Global p95 200ms"
      "10x burst / 5min"
    Availability
      "99.95% SLO"
      "Multi‑Region resilience"
    Security & DRM
      "Signed URLs"
      "Anomaly detection"
      "Token-based entitlements"
    Cost Efficiency
      "Egress optimization"
      "Storage tiering"
    Interoperability
      "Coexist with legacy"
    Observability
      "SLO dashboards"
      "Tracing"
      "Automated insights"
```

---

## 3. Key Architecture Decision (ADR)

### ADR‑001: Use S3 + CloudFront for global content distribution with serverless entitlements and DynamoDB Global Tables.

### High‑Level Architecture (Mermaid Diagram)
```mermaid
flowchart LR
    subgraph Client["Global Users (B2C & B2B)"]
        A["Web/Mobile/E‑Readers"]
    end

    A --> CF["CloudFront CDN"]
    CF --> EDGE["Lambda@Edge / CloudFront Functions<br/>Token Validation, DRM Enforcement"]

    EDGE --> OS["Origin Shield"]
    OS --> S3["S3 Buckets<br/>(Cross‑Region Replication)"]

    A --> API["API Gateway"]
    API --> L["Lambda Entitlement Service"]
    L --> DDB["DynamoDB Global Tables<br/>Entitlements / Metadata"]

    subgraph Observability
        CW["CloudWatch Metrics & Logs"]
        XR["X‑Ray Tracing"]
        CAD["Cost Anomaly Detection"]
    end

    CF -.-> CW
    L -.-> CW
    API -.-> XR
```

---

## 4. Architectural Alternatives & Trade‑offs

### Alternatives Comparison Diagram
```mermaid
graph TD
    A["CloudFront + S3 + Serverless Entitlement (Chosen)"]
    B["Multi‑Region ECS/EKS Delivery"]
    C["DynamoDB Global Tables + Pre‑Signed URLs"]
    D["Real‑Time DRM Transformation Gateway"]

    A -->|Pros: Best performance & cost| Z["Selected"]
    B -->|Cons: High ops & warmup issues| Z
    C -->|Pros: Great for metadata lookup| Z
    D -->|Cons: Too slow for global p95| Z
```

---

## 5. GenAI-Assisted Operational Enhancements

### GenAI Operational Flow Diagram
```mermaid
sequenceDiagram
    autonumber
    participant M as Metrics (CloudWatch/CDN)
    participant G as GenAI Model (Forecasting & Insights)
    participant EB as EventBridge
    participant AWS as AWS Services

    M->>G: Provide demand, cost, anomalies
    G->>EB: Predictive scaling & optimization recommendations
    EB->>AWS: Adjust scaling, TTL, capacities, warm pools
    AWS->>M: Updated metrics for next cycle
```

---

## 6. Stakeholder Impact

### Impact Diagram
```mermaid
flowchart TB
    subgraph Business
        B1["Faster time‑to‑market"]
        B2["SLA compliance"]
        B3["Revenue uplift"]
        B4["Predictable cloud spend"]
    end

    subgraph Technical
        T1["Simplified architecture"]
        T2["Strong DRM posture"]
        T3["Lower ops overhead"]
        T4["Global resilience"]
    end

    B1 --- T1
    B2 --- T2
    B3 --- T3
    B4 --- T4
```

---

## 7. Final Notes & Next Steps
- Run performance tests (p95, burst scaling)  
- Validate DRM enforcement at edge  
- Enable GenAI pipeline for operations  
- Prepare production rollout plan  

---
