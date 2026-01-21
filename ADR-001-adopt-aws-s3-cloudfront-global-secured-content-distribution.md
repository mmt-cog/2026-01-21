# ADR-001: Deliver global, low-latency DRM-protected digital content for B2C/B2B under strict scale, availability, and cost constraints

**Business problem (Digital Transformation & Modernization):** Redesign the product’s publishing and delivery platforms to reliably ship DRM-protected e-books and interactive media at global scale across multi-channel distribution.

**Pain points (business impact):**
- There is no web 
- Global readers experience latency that degrades UX and conversion without a sub-200 ms delivery tier.
- Marketing launches and releases trigger traffic spikes that risk slowdowns/outages without 10× burst elasticity.
- Availability gaps directly impact revenue and B2B SLAs without 99.95% resilience.
- Weak entitlement/DRM controls increase piracy, contract risk, and partner friction.
- High egress and origin load can make unit economics unpredictable without aggressive caching and cost guardrails.




## Decision
Implement a global digital delivery tier on AWS:

- **CDN & Edge Security:** Use **Amazon CloudFront** with **Signed URLs/Cookies**, **Origin Shield**, and **edge functions** for lightweight token validation.
- **Content Storage:** Store digital content in **Amazon S3**, with **cross-Region replication**.
- **Entitlement/DRM APIs:** Build entitlement/DRM services using **API Gateway + AWS Lambda**.
- **Global Entitlement Metadata:** Use **DynamoDB Global Tables** for globally distributed entitlement metadata.
- **Identity:** Use **Amazon Cognito** for B2C, with **SAML/OIDC federation** for B2B.
- **Operations Automation:** Use **EventBridge** to drive predictive autoscaling and operational workflows.
- **Observability & Cost Controls:** Use **CloudWatch**, **X-Ray**, **AWS Budgets**, and **Cost Anomaly Detection**.

## Consequences
### Pros
- Edge caching achieves sub-200 ms latency
- Highly elastic with minimal ops overhead
- Strong, scalable DRM/entitlement model
- Cost-efficient for high-read workloads (when cache hit ratio is high)

### Cons / Risks
- Token lifecycle complexity (issuance, expiry, refresh, revocation)
- Edge DRM logic must remain lightweight (latency + execution limits)
- Egress remains a cost-sensitive dimension and must be actively managed

## Phase 1 implementation plan (Digital delivery tier)

### 1) Scope and success criteria
- Define Phase 1 scope: digital asset delivery + entitlement checks; coexistence with legacy editorial/distribution systems.
- Establish measurable targets and dashboards:
  - p95 global read latency ≤ 200 ms (by region)
  - Cache hit ratio target (by path/class of asset)
  - Entitlement API p95 latency target and error budget
  - Availability SLOs (CDN + API)
  - Cost guardrails (monthly spend thresholds; egress alerts)

### 2) Content storage, replication, and access model (S3)
- Create S3 buckets:
  - Primary content bucket (Region A)
  - Replica content bucket (Region B) with **CRR**
- Enable:
  - Versioning, lifecycle policies (as required by retention)
  - Encryption at rest (KMS-managed)
- Define S3 prefix conventions for content classes (e.g., books, covers, media, manifests).
- Configure origin access so S3 is not publicly accessible and is only reachable via CloudFront.

Deliverables:
- Bucket/IaC definitions, replication configuration, KMS keys/policies
- Documented object key scheme and content publishing workflow

### 3) CDN distribution (CloudFront)
- Create CloudFront distribution with:
  - S3 origin configured with **Origin Shield**
  - Cache policies per content class (TTL, cache key choices)
  - Signed URLs/Cookies for DRM-protected assets
- Add edge logic for lightweight checks:
  - Validate presence/shape of access token or signed cookie/URL
  - Enforce basic constraints (expiry, required claims, path binding)
  - Keep logic minimal; all heavy decisions remain in APIs

Deliverables:
- CloudFront distribution, behaviors, cache/origin request policies
- Key management approach for signed URLs/cookies (rotation + operational process)
- Documented CDN caching strategy (what is cacheable vs always-authenticated)

### 4) Entitlement and DRM APIs (API Gateway + Lambda)
- Define API surface for Phase 1 (minimum viable set):
  - `POST /entitlements/issue` (issue short-lived access token / signed URL/cookie)
  - `POST /entitlements/validate` (server-side validation for clients that require it)
  - `POST /drm/license` (if required for the chosen DRM flow)
- Implement Lambda functions:
  - Integrate with Cognito for identity and token verification
  - Apply business rules for entitlements (B2C purchases, subscriptions, B2B rights)
  - Generate signed URLs/cookies or short-lived access tokens
  - Emit audit/ops events (successful issuance, denials, anomalies)

Deliverables:
- OpenAPI definition (or equivalent) for the Phase 1 APIs
- Lambda code + deployment configuration
- Rate limits/throttling settings consistent with burst requirements

### 5) Entitlement metadata store (DynamoDB Global Tables)
- Design data model for global entitlement reads:
  - Partition key strategy to avoid hot partitions
  - TTL for ephemeral/session entitlements
  - Conditional writes for consistency (grant/revoke)
- Enable DynamoDB Global Tables to support low-latency validation globally.

Deliverables:
- Table schema (attributes, TTL, GSIs if needed)
- Replication regions and capacity/scaling approach

### 6) Identity (Cognito + federation)
- Configure Cognito:
  - B2C user pool/app clients
  - Federation via SAML/OIDC for B2B
- Decide token claims required for entitlement decisions (tenant/customer, entitlements scope, device binding signals).

Deliverables:
- Cognito configuration and federation setup documentation
- Claim contract for access and entitlement tokens

### 7) Event-driven operations (EventBridge)
- Define event types and producers/consumers for Phase 1:
  - Traffic forecast / release events
  - Token abuse/anomaly signals
  - Operational workflows (scale readiness checks, on-call notifications)
- Use EventBridge rules to drive:
  - Pre-scaling actions where applicable
  - Automated runbooks/notifications

Deliverables:
- Event schema definitions and routing rules
- Operational workflows for key events

### 8) Observability and cost controls
- Observability:
  - CloudWatch dashboards for CDN + API + DynamoDB
  - X-Ray tracing for entitlement/DRM API calls
  - SLO/alerting tied to p95 latency and error rates
- Cost controls:
  - AWS Budgets with alert thresholds
  - Cost Anomaly Detection configured for egress and CDN spend patterns

Deliverables:
- Dashboards, alarms, and runbooks
- Budget thresholds and response playbook

### 9) Validation, security review, and launch
- Validate:
  - Performance under burst (load tests hitting CDN + entitlement APIs)
  - Cache correctness (private vs public assets; token-bound paths)
  - Token lifecycle correctness (expiry, refresh, revocation behavior)
  - Regional failover behavior (replication + global tables)
- Launch plan:
  - Pilot rollout (subset of catalog/regions)
  - Gradual traffic ramp with rollback criteria

Deliverables:
- Test plan + results
- Cutover checklist and rollback plan

## Open questions (to resolve during Phase 1)
- Token strategy: Signed URLs vs signed cookies vs short-lived JWT-like access tokens (and how each maps to client types).
- Revocation approach: immediate vs eventual consistency (and what user experience is acceptable).
- Device binding signals and privacy/compliance requirements.
- Content classes that can be safely cached long-lived vs must be short-lived.
- Egress governance: target cache hit ratio by asset class and acceptable miss rate during launches.
