# PCI Tokenization Service Requirements

## Overview

Jack Henry Associates operates a central PCI DSS 4.0-compliant tokenization platform on Google Cloud Platform (GCP). This service is consumed exclusively by Jack Henry internal application services — Banno, SilverLake, Symitar, ProfitStars, and others — acting on behalf of bank and credit union institution clients.

The service tokenizes Primary Account Numbers (PANs), maintains a secure token vault, supports multiple tokens per PAN scoped by institution and caller context, and provides robust audit logging and compliance evidence for PCI DSS 4.0 requirements.

## Architecture

1. **Infrastructure**
   - **Platform**: GCP. Compute layer: GKE private cluster with VPC-native pod networking. VPC-native networking satisfies PCI DSS 1.2/1.3 network segmentation requirements without Customized Approach.
   - **API Gateway**: Google Cloud API Gateway or Apigee for ingress, request routing, and rate limiting.
   - **Token Vault**: Cloud Spanner (partitioned by `institution_id`) for token and card record storage. AES-256-GCM field-level encryption at the application layer.
   - **Cryptographic Keys**: Cloud KMS with HSM backing. Two key classes per institution: PFK (HMAC-SHA-256 for PAN fingerprinting, institution-scoped) and PEK (AES-256-GCM envelope encryption, institution-scoped). Both provisioned at institution onboarding.
   - **Cache Layer**: Redis (hot token lookups, keyed by `institution_id` as leading dimension; ≥ 85% hit rate target).
   - **Audit Sink**: Cloud Pub/Sub → BigQuery for immutable, queryable audit records.
   - **Logging & Monitoring**: Cloud Logging, Cloud Monitoring, Cloud Trace. Per-institution and per-caller-application dashboards.

2. **Institution Model**
   - **Institution** (`institution_id`): A bank or credit union client of Jack Henry. The primary data isolation boundary. All vault data, cache keys, audit events, and cryptographic keys are partitioned by `institution_id`.
   - **Caller Application**: A Jack Henry internal service (e.g., `banno-payments@jh.iam`). Authenticated via Google Cloud Workload Identity. One application may serve thousands of institutions.
   - **Delegation Authorization**: A grant table mapping `(caller_app_identity, institution_id)` to permitted operations, domains, and purposes. A caller application must be explicitly authorized for each institution it serves.
   - **Institution Registry**: Maps `institution_id` to Cloud KMS key references (PFK key name, PEK key name). Stored in a dedicated Spanner config table (IAM-isolated from vault data tables). In-process cached with 60s TTL. Required to exist before any API call for an institution succeeds.

3. **Token Logic**
   - Support multiple tokens per PAN, differentiated by institution, domain, and `scope_qualifiers` (a key-value map for JH application/channel context, e.g., `{"application": "banno-mobile", "channel": "web"}`).
   - REUSABLE mode: returns an existing ACTIVE token if one exists for the scope `(institution_id, domain, scope_qualifiers_canonical, purpose, pan_fingerprint)`.
   - ONE_TIME mode: always mints a new token, regardless of existing tokens for the scope.
   - Tokens are revocable by token value or by institution fingerprint selector with optional `scope_qualifiers` filter.
   - Token lifecycle: ACTIVE → REVOKED (terminal); EXPIRED for TTL-elapsed tokens.

4. **Security & Compliance**
   - **PCI DSS 4.0** adherence: field-level AES-256-GCM encryption (Req 3.5.1.2), institution-scoped HMAC-SHA-256 PAN fingerprinting (Req 3.5.1.1), Cloud KMS HSM-backed key management, 90-day key rotation, key/data separation of duties at IAM.
   - **Authentication**: Google Cloud Workload Identity for all JH caller applications. No static API keys or long-lived credentials.
   - **Authorization**: Four-step chain — Workload Identity validity → delegation grant for `institution_id` → domain/purpose policy → (detokenize full-PAN only) `full-pan` IAM scope gate.
   - **Detokenize default**: Returns masked PAN (`411111******1111`). Full PAN requires `full-pan` IAM scope.
   - **Fail-closed**: Any authorization failure, delegation grant service error, or policy uncertainty results in denial. No fallback to allow.
   - **Audit logging**: Immutable audit events with `caller_application` and `institution_id` as mandatory indexed fields. Reason code and (optional) operator_id recorded on detokenize.
   - **MFA**: Required for all Jack Henry personnel with CDE resource access. Enforced at Google Workspace IdP level (PCI DSS Req 8.4.2).
   - **PAN discovery**: Automated scanning across all CDE-adjacent storage annually and after infrastructure changes (PCI DSS Req 12.5.2).
   - **Vulnerability scanning**: Three-tier model — unauthenticated external, unauthenticated internal, authenticated internal quarterly (PCI DSS Req 11.3.1.2).

5. **Operational Requirements**
   - **Throughput & Latency**: 50k RPS peak; tokenize p95 ≤ 30ms, detokenize p95 ≤ 35ms.
   - **Rate limiting**: Per-institution (500 RPS default, 2,000 burst, 10,000 hard ceiling) and per-caller-application (5,000 RPS default). Buckets are institution-isolated.
   - **Availability**: 99.95% monthly SLO for tokenize and detokenize APIs.
   - **Data durability**: 99.999999999% for token vault metadata.
   - **RTO ≤ 30 minutes; RPO ≤ 5 minutes** (single-region with multi-zone HA; multi-region active-active is a Future Phase).
   - **Deployments**: Canary strategy with per-institution traffic analysis and automatic rollback.

6. **APIs**
   - `POST /v1/tokenize`: Issue a token for a PAN scoped to an institution, domain, and optional `scope_qualifiers`.
   - `POST /v1/detokenize`: Resolve a token to PAN. Requires `institution_id`, `reason_code`. Returns masked PAN by default.
   - `POST /v1/tokens/revoke`: Revoke by token value or by `(institution_id, domain, pan_fingerprint, scope_qualifiers_filter)`.
   - All requests require valid Workload Identity token + delegation grant for the supplied `institution_id`.
   - No public-facing SDKs; this is an internal Jack Henry service. OpenAPI spec maintained for internal consumers.

7. **Institution Onboarding**
   - Before an institution can be processed:
     1. Provision institution-scoped PFK in Cloud KMS (HMAC-SHA-256, HSM-backed, 90-day rotation).
     2. Provision institution-scoped PEK in Cloud KMS (AES-256-GCM, HSM-backed, 90-day rotation).
     3. Register institution in the Institution Registry (`institution_id` → PFK key name, PEK key name).
     4. Configure delegation grants for authorized JH caller applications.
     5. Run end-to-end smoke test (tokenize → detokenize → revoke) with synthetic PAN.
     6. Set institution status to `active`.
   - Institution Shared Responsibility Agreement executed with each institution client via Jack Henry Legal before or at production enablement.

8. **Compliance Governance**
   - Quarterly executive-level PCI compliance reviews covering open control gaps and institution obligations (PCI DSS Req 12.4.1).
   - Institution Shared Responsibility Agreement specifies Jack Henry-owned controls vs. institution-owned controls (PCI DSS Req 12.9.1).
   - Customized Approach framework applied to any controls deviating from PCI DSS prescriptive requirements; documented with QSA acceptance.

10. **Compute Layer Migration Path: Cloud Run → GKE**

   The service is packaged as a Docker container. This enables a two-phase deployment strategy: start on Cloud Run for speed and operational simplicity, then migrate to GKE private cluster when PCI compliance posture, institution scale, or operational maturity requires it. The application code and Docker image do not change between phases — only the infrastructure layer changes.

   **Phase 1 — Cloud Run (Move Fast)**
   - Deploy the Docker container as a Cloud Run service with VPC connector for private network access to Spanner, Redis, and KMS.
   - Cloud Run provides built-in autoscaling, zero-config TLS termination, and Workload Identity support.
   - **PCI DSS network segmentation caveat**: Cloud Run does not provide a two-way configurable VPC firewall at the pod/container level, which creates a gap against PCI DSS 1.2/1.3. Before production CDE deployment on Cloud Run, the team must:
     1. Document compensating controls at the API Gateway and VPC connector layer.
     2. Classify the approach under the PCI DSS **Customized Approach** framework (not Compensating Controls — see SR-02a in detailed requirements).
     3. Obtain QSA acceptance of the Customized Approach documentation before first institution goes live.
   - All other PCI controls (encryption, Workload Identity, audit logging, key management, MFA) are identical between phases — Cloud Run does not weaken them.
   - Phase 1 is appropriate for: development, integration testing, non-production environments, and early production with QSA-accepted Customized Approach in place.

   **Phase 2 — GKE Private Cluster (Compliance Hardened)**
   - Migrate the same Docker image to GKE private cluster with VPC-native pod networking.
   - GKE Network Policy restricts pod-to-pod and pod-to-internet traffic to authorized paths only.
   - VPC-native pod networking maps directly to PCI DSS 1.2/1.3 — the Customized Approach documentation from Phase 1 is retired.
   - Migration effort is **infrastructure-only**: the Docker image, application code, Workload Identity bindings, API contract, KMS key references, Spanner schema, and Redis configuration are all unchanged.

   **What changes between Phase 1 and Phase 2:**

   | Layer | Cloud Run (Phase 1) | GKE (Phase 2) |
   | --- | --- | --- |
   | IaC | `google_cloud_run_service` Terraform resources | GKE cluster, node pools, Deployment, Service, Ingress, HPA |
   | Networking | VPC connector; no pod-level firewall | VPC-native pods; Network Policy; full firewall control |
   | Autoscaling | Cloud Run concurrency-based (automatic) | Kubernetes HPA (CPU/RPS targets, configurable) |
   | Workload Identity | Cloud Run service account annotation | GKE `serviceAccountAnnotation` on K8s ServiceAccount |
   | PCI DSS 1.2/1.3 | Requires Customized Approach + QSA acceptance | Satisfied natively; no Customized Approach needed |
   | Cold-start behavior | Managed by Cloud Run min-instances setting | GKE always-warm pods (no cold starts at min replicas) |

   **Migration trigger criteria** — migrate from Phase 1 to Phase 2 when any of the following is true:
   - QSA requires GKE for network segmentation compliance before scope expansion.
   - Institution count exceeds 50 (operational scale warrants GKE management investment).
   - Platform team is ready to own GKE cluster lifecycle (upgrades, node pools, Network Policy).
   - Customized Approach maintenance overhead exceeds GKE migration cost.

9. **KMS Key Scale Management**
   - With both PFK and PEK institution-scoped, Cloud KMS key count grows linearly with institution count (2 keys per institution).
   - The onboarding pipeline (E-16) shall enforce key naming conventions and track provisioned keys in the Institution Registry. Orphaned keys (institution offboarded but keys not destroyed) shall be detected by a scheduled reconciliation job.
   - GCP Cloud KMS quotas to monitor as institution count grows:
     - Key rings per project (default: 1,000 — consider one key ring per institution or a small number of shared key rings with distinct key names).
     - Cryptographic operations per second per key (default: 600 sign/verify ops/s per key — size key rings and key distribution accordingly at high institution + high RPS scale).
   - At institution counts above 500, the platform team shall review key ring topology and KMS quota headroom before onboarding additional institutions.
   - Key rotation across a large institution fleet shall be pipeline-automated; manual rotation at scale is not operationally viable and is a compliance risk.

---

This document is the top-level architectural specification. Detailed functional requirements are in `docs/pci-tokenization-requirements.md`. API contract is in `docs/api-contract.md`. Epics and stories are in `agile/epics.md` and `agile/stories.md`.
