# PCI Tokenization Service Requirements (Scale)

## 1. Purpose

Define requirements for a high-volume, low-latency PCI tokenization platform on GCP with Python services, including:
- token vault storage for sensitive card data
- multiple tokens per card by domain, scope qualifiers, and purpose
- controlled de-tokenization for authorized services with mandatory audit context

This document is intended as the baseline for architecture, implementation, security controls, and production readiness. It aligns with [Google Cloud’s architecture guide for tokenizing sensitive cardholder data for PCI DSS](https://docs.cloud.google.com/architecture/tokenizing-sensitive-cardholder-data-for-pci-dss) and the [PCI DSS Tokenization Guidelines](https://www.pcisecuritystandards.org/documents/Tokenization_Guidelines_Info_Supplement_v2_0.pdf).

## 2. Scope

### In Scope

- Tokenization API for PAN to token conversion.
- De-tokenization API for token to PAN retrieval with strict authorization.
- Token revocation API.
- Vault data model and cryptographic key lifecycle controls.
- Audit logging and compliance evidence generation.
- Single-region high availability deployment.
- Load/performance validation up to 50k TPS class.

### Out Of Scope

- Direct cardholder-facing UI.
- Payment gateway orchestration.

### Future Phase (Planned)

- **Multi-region active-active architecture**: Active-active deployment across two or more U.S. regions with global token vault consistency, cross-region PFK/PEK replication via Cloud KMS, and sub-5-minute RPO on regional failure. Initial delivery is single-region; multi-region is a planned follow-on phase requiring separate architecture and DR requirements. See §15 Risks for single-region dependency risk acknowledgment.

## 3. Stakeholders

- Payments Platform Engineering
- Security Engineering
- Compliance / GRC
- Site Reliability Engineering
- Tenant Integrations / Internal API Consumers

## 4. Definitions

- **PAN**: Primary Account Number.
- **CHD**: Cardholder Data (card data that PCI DSS Part 3 requires to be protected).
- **CDE**: Cardholder Data Environment; systems that process, store, or transmit CHD.
- **Token**: A surrogate value substituted for sensitive data (e.g., PAN). A token is meaningful only as a lookup key in a specific context. Per PCI tokenization guidance and GCP architecture: tokens must not contain user-specific information and must not be directly decryptable, so that loss of tokens does not compromise cardholder data.
- **Domain**: Namespace for token scoping (e.g., checkout, subscription, in-store). Distinct from merchant or application identity.
- **Scope Qualifier**: An optional caller-supplied label for further differentiating token issuance within a domain (e.g., merchant ID, application ID, channel). Multiple scope qualifiers may be provided as a key-value map. See FR-01 and FR-02.
- **Purpose**: Allowed business function (payment, refund, etc.).
- **Reusable Token**: Stable token for same `(tenant, domain, scope_qualifiers, purpose, PAN fingerprint)`.
- **One-Time Token**: Ephemeral token minted on each request.
- **PAN Fingerprint Key (PFK)**: A tenant-scoped HMAC secret key used exclusively as keying material for deterministic PAN fingerprint computation. The PFK is distinct from the PAN Encryption Key (PEK) and shall never be used for encryption or decryption. Both key classes are managed in Cloud KMS per SR-01 and SR-03.
- **PAN Encryption Key (PEK)**: The envelope encryption key used to encrypt PAN payloads before storage in the vault. Managed in Cloud KMS with HSM-backed protection.

## 5. Functional Requirements

### FR-01 Tokenize API

- System shall provide `POST /v1/tokenize`.
- Caller identity (tenant) shall be resolved server-side from the authenticated Workload Identity token. Callers shall not supply `tenant_id` in the request body.
- Required inputs: `domain`, `token_purpose`, `token_mode`, PAN payload, `idempotency_key`.
- Optional inputs: `scope_qualifiers` (key-value map, e.g., `{"merchant_id": "m_99821", "application_id": "app-mobile-v2"}`), `ttl_seconds`.
- System shall support:
  - `REUSABLE` mode: return existing active token for matching `(tenant, domain, scope_qualifiers, token_purpose, pan_fingerprint)` when available.
  - `ONE_TIME` mode: always mint a new token regardless of existing tokens.
- System shall return token metadata including token value, mode, state, and optional expiry.

### FR-02 Multi-Token Per Card

- System shall support multiple active tokens for one PAN across:
  - different domains
  - different scope_qualifiers within a domain (e.g., different merchants, applications, or channels)
  - different purposes
  - one-time issuance
- The reusable token uniqueness key is `(tenant, domain, scope_qualifiers_canonical, purpose, pan_fingerprint)`. `scope_qualifiers` are canonicalized as lexicographically sorted key-value pairs before lookup and storage.
- System shall prevent token value collisions globally across all tenants.

### FR-02a Token Security (PCI / GCP Alignment)

- Tokens shall not contain any user-specific or cardholder-identifying information.
- Tokens shall not be directly decryptable to recover PAN without access to the secured vault and authorized de-tokenization path.
- Design shall ensure that compromise or loss of tokens alone cannot be used to derive or access cardholder data.

### FR-03 De-Tokenize API

- System shall provide `POST /v1/detokenize`.
- Required inputs: `domain`, `token_purpose`, `token`, `request_context.reason_code`. Caller identity resolved from Workload Identity token.
- Optional inputs: `scope_qualifiers`, `request_context.transaction_id`, `request_context.operator_id`, `format_options.return_type`.
- System shall require caller identity and policy checks before PAN release. Policy must validate service identity + domain + scope_qualifiers + purpose.
- System shall fail closed on any authorization uncertainty or policy service error — no degraded allow path.
- System shall return **masked PAN by default** (`411111******1111`). Full PAN shall only be returned when `format_options.return_type` is `FULL_PAN` and the caller holds the `full-pan` IAM scope. Absent the scope, the request shall be rejected with `403 FORBIDDEN`.
- `request_context.reason_code` shall be a required enumerated field (e.g., `PAYMENT_PROCESSING`, `REFUND_PROCESSING`, `FRAUD_INVESTIGATION`, `CHARGEBACK`, `SETTLEMENT`, `COMPLIANCE_REVIEW`). Requests without a valid reason code shall be rejected.
- `request_context.operator_id`, when provided, is caller-asserted audit metadata. It shall be written to the audit event but is not verified by the tokenization service; it shall be marked as unverified in the audit record.

### FR-04 Token Revocation

- System shall provide `POST /v1/tokens/revoke`.
- System shall support revocation by:
  - explicit token value
  - fingerprint selector: `(tenant, domain, pan_fingerprint)` with optional `scope_qualifiers` filter. When `scope_qualifiers` are provided, revocation is restricted to tokens matching both the domain and all supplied qualifier key-value pairs. When omitted, all tokens for the fingerprint within the domain are revoked.
- A required `reason` field shall be provided for all revocation requests (e.g., `merchant_request`, `fraud_signal`, `card_compromised`, `merchant_offboarded`, `application_decommissioned`, `compliance_action`).
- Revoked tokens shall not be de-tokenizable. Revocation is permanent and not reversible via API.

### FR-05 Idempotency

- Tokenize requests shall be idempotent by `(caller_identity, idempotency_key)`. Tenant is resolved from caller identity.
- Replays with same idempotency key shall return the original successful result without re-minting or re-encrypting.
- Detokenize is a read operation; true write idempotency does not apply. However, duplicate detokenize requests carrying the same `x-request-id` within a 60-second deduplication window shall return a cached response without emitting a second audit event.

### FR-06 Auditability

- Every tokenize, detokenize, revoke, and denied request shall emit an audit event.
- Audit events shall include: request_id, caller_identity (actor), action, outcome, timestamp, domain, scope_qualifiers, token_purpose.
- Detokenize audit events shall additionally include: reason_code, operator_id (marked unverified if caller-asserted), return_type (MASKED or FULL).
- Revoke audit events shall additionally include: revocation selector type (token or fingerprint), reason, and revoked_count.
- Audit events shall never include PAN, CVV, key material, or full token values.

## 6. Data Requirements

### DR-01 Vault Data Model

- Vault shall maintain card records and token records with logical linkage. Card records store the AES-256-GCM encrypted PAN and expiry. Token records store the token value, state, domain, scope_qualifiers, purpose, token_mode, and a reference to the card record.
- PAN shall never be stored plaintext. Field-level encryption is applied at the application layer before any write per SR-01.
- PAN fingerprint shall be computed using HMAC-SHA-256 with the tenant-scoped PFK per SR-01. The fingerprint is the indexed lookup key for card record retrieval and reusable token deduplication.
- Token records shall include a canonicalized `scope_qualifiers` representation (sorted key-value pairs) to support deterministic lookup.
- Primary index for reusable token lookup: `(tenant_id, domain, scope_qualifiers_canonical, purpose, pan_fingerprint, state)`.
- Secondary index for fingerprint-based revocation: `(tenant_id, domain, pan_fingerprint, state)` with optional scope_qualifiers filter at query time.
- `scope_qualifiers` keys and values shall be validated as non-sensitive strings; PAN-like values in qualifier fields shall be rejected at input validation.

### DR-02 Data Retention

- Audit retention periods shall meet PCI and enterprise policy.
- Token and card metadata retention shall align with contractual and legal obligations.

### DR-03 Data Residency

- Data shall remain in configured single region (initial phase).

## 7. Security And PCI Requirements

### SR-01 Encryption

- CHD shall be encrypted in transit using TLS 1.2 minimum; TLS 1.3 is preferred.
- CHD shall be encrypted at rest using **field-level (application-layer) envelope encryption** with AEAD (AES-256-GCM or equivalent). The minimum symmetric key length is 256 bits per PCI DSS 3.4.1.
- Disk-level, partition-level, or storage-layer encryption (e.g., Spanner's default at-rest encryption, Cloud Storage default encryption) does not satisfy the PAN unreadability requirement and shall not be used as the sole protection mechanism per PCI DSS 3.5.1.2. Application-layer encryption must be applied before writes to any storage system.
- PAN fingerprint shall be computed using **HMAC-SHA-256** with a dedicated tenant-scoped PAN Fingerprint Key (PFK). Unkeyed hashes (e.g., plain SHA-256 of the PAN) are prohibited per PCI DSS 3.5.1.1 because the bounded PAN digit space is vulnerable to brute-force correlation.
- Two distinct key classes shall be maintained in Cloud KMS and shall never be used interchangeably:
  - **PAN Encryption Key (PEK)**: Used for envelope encryption of stored PAN payloads. Grants: `roles/cloudkms.cryptoKeyEncrypterDecrypter`.
  - **PAN Fingerprint Key (PFK)**: Used only for HMAC-based PAN fingerprint computation. Grants: a separate, scoped KMS key with MAC sign/verify permissions.
- Both PEK and PFK shall use HSM-backed Cloud KMS key rings in production per PCI DSS 3.6.
- KEKs shall be rotated per SR-03; PAN Fingerprint Keys shall follow the same rotation cadence.

### SR-02 Access Control

- Access shall be least-privilege and identity-based. A dedicated service account shall be used for the tokenization service (not the default Cloud Run/Compute Engine service account); it shall have only the roles necessary (e.g., Cloud KMS CryptoKey Encrypter/Decrypter) to operate the tokenizer and detokenizer.
- All API access shall require authenticated caller identity (e.g., OAuth2/identity token); unauthenticated access shall be disabled (`no-allow-unauthenticated` / equivalent). Callers must hold invoke permission (e.g., Cloud Run Invoker or equivalent) to call tokenize/detokenize.
- De-tokenization shall require explicit permission by service identity + domain + scope_qualifiers + purpose.
- Administrative break-glass access shall require dual control.
- **Hard-coded credentials are prohibited.** Service account credentials, signing keys, HMAC keys, and any secret material shall never appear in application source code, configuration files, container images, Dockerfiles, CI/CD pipeline definitions, or version control. Violation shall be treated as a critical security incident requiring immediate rotation and incident review.
- The tokenization service shall authenticate to GCP services using **Google Cloud Workload Identity** (or equivalent short-lived credential mechanism). Long-lived static service account key files shall not be used for the production tokenization service. If key files are required in any non-production context, they shall be rotated at least every 90 days, stored in Secret Manager, and access-logged.
- Static service account key usage shall be monitored via Cloud Audit Logs; any key use from an unexpected compute context or IP range shall trigger an alert per OR-02.

### SR-02a Network And CDE (PCI DSS 1.2, 1.3)

- In a PCI DSS compliant CDE, inbound and outbound traffic shall be limited to authorized connections. PCI DSS requirements 1.2 and 1.3 require tight controls on network traffic.
- Serverless runtimes (e.g., Cloud Run, App Engine) do not offer a two-way configurable firewall. Therefore the organization shall either:
  - implement and document compensating controls at the application and gateway layer, and obtain QSA/SAQ acceptance; or
  - deploy the tokenization service on a platform with full VPC controls (e.g., GKE, Compute Engine managed instance groups) for production CDE.
- Documented architecture and control set shall be acceptable to the Qualified Security Assessor or Self-Assessment Questionnaire process.

### SR-03 Key Management

- Keys shall have defined rotation cadence (target 90 days); Cloud KMS rotation period shall be configured accordingly. This applies to both PEK and PFK key classes.
- Key usage shall be logged and monitored for anomaly detection.
- **Separation of duties between key management and data access shall be enforced at the IAM level.** IAM roles granting key management capabilities (e.g., `roles/cloudkms.admin`, `roles/cloudkms.cryptoKeyVersionCreator`, `roles/cloudkms.cryptoKeyVersionDestroyer`) shall not be granted to any principal that also holds data-plane access to Spanner vault tables, Secret Manager secrets containing CHD, or Redis cache data.
- The runtime tokenization service account shall hold only the minimum KMS roles required for runtime encrypt/decrypt operations (`roles/cloudkms.cryptoKeyEncrypterDecrypter` for PEK; MAC sign/verify for PFK). It shall never hold KMS administrative roles.
- Key management administrative access (rotation, import, destruction) shall be restricted to a designated key custodian identity separate from both the service runtime identity and database administrator identities.
- Compliance with separation of duties shall be validated quarterly via IAM policy snapshot review, producing an evidence artifact per SR-05.

### SR-04 Logging And Redaction

- PAN/CVV shall never be present in logs, traces, or error bodies.
- Structured logging shall enforce sensitive field redaction.

### SR-05 PCI Evidence

- System shall produce artifacts for audits:
  - access review records (monthly IAM access reviews)
  - key rotation logs (PEK and PFK rotation events from Cloud KMS)
  - key/data separation of duties validation (quarterly IAM snapshot review)
  - security test evidence
  - incident and change records
- **Internal vulnerability scans shall be conducted quarterly using authenticated scanning methods per PCI DSS 11.3.1.2.** Authenticated scans shall use credentials with sufficient access to assess OS patch levels, running service versions, and application-layer configuration — not only network-layer port scanning. Evidence shall distinguish between the three scan tiers:
  - **(a) Unauthenticated external scans** — network perimeter, per PCI DSS 11.3.2.
  - **(b) Unauthenticated internal scans** — internal network layer, per PCI DSS 11.3.1.
  - **(c) Authenticated internal scans** — OS and application layer with credentialed access, per PCI DSS 11.3.1.2. This tier is required and shall not be substituted by (a) or (b).
- Scan results and remediation records shall be maintained in the compliance evidence repository with timestamps and responsible-party attribution.

### SR-06 Dependencies And Supply Chain (GCP Production Guidance)

- Dependencies (e.g., Python packages) shall be pinned to specific, vetted versions. Versions shall be bundled with the application or sourced from a private, trusted repository.
- This approach reduces risk of downtime from public repository outages and from supply-chain attacks on packages assumed to be safe. Pre-built, bundled deployments are preferred to reduce deployment time and improve launch/scaling consistency.

### SR-07 Multi-Factor Authentication For CDE Human Access (PCI DSS 4.0 Req 8.4.2, 8.3.6)

- All human access to Cardholder Data Environment (CDE) resources shall require multi-factor authentication (MFA), regardless of network origin. This requirement applies to all personnel including developers, SREs, security engineers, contractors, and third-party vendors.
- CDE resources requiring MFA include but are not limited to: GKE cluster management interfaces, Spanner data-plane access, Cloud KMS administration consoles, Secret Manager access, Cloud Console for production CDE-scoped projects, and any SSH or API-based access to CDE-scoped compute resources.
- MFA shall be enforced at the identity provider level (e.g., Google Workspace 2-Step Verification enforcement policy) and shall not rely on individual user opt-in. Users shall not be able to bypass MFA enforcement.
- Break-glass emergency access procedures shall include MFA as a mandatory step. Break-glass shall not provide a path to bypass MFA; if MFA is unavailable, the break-glass procedure itself shall be blocked, and escalation to dual-control manual procedure applies.
- If password-based authentication is used as one factor, passwords shall be a minimum of **12 characters** in length, meeting complexity requirements per PCI DSS 8.3.6. Password-only single-factor access to any CDE resource is prohibited.
- MFA enforcement shall be validated as part of each release readiness review and included in the quarterly IAM access review evidence.

### SR-08 Automated PAN Data Discovery (PCI DSS 4.0 Req 12.5.2)

- Automated tooling shall scan all accessible data storage locations within or adjacent to the CDE boundary for the presence of unprotected or out-of-place PANs. Scan targets shall include at minimum: Cloud Logging export buckets, BigQuery audit tables, Cloud Error Reporting payloads, Spanner table exports, Redis cache snapshots, CI/CD artifact storage, and any developer-accessible non-production environments that may receive real data.
- PAN data discovery scans shall be executed at minimum every **12 months** and additionally after any significant infrastructure change (e.g., new log sinks, new storage buckets, data pipeline modifications, or schema changes).
- Discovery tool results shall be reviewed within 30 days of scan completion. Any confirmed PAN presence outside the designated vault boundary shall be classified as a potential data exposure incident and handled per OR-04 incident procedures.
- SR-04 (log redaction) is a preventive control; SR-08 (PAN discovery) is a detective control. Both are required — one does not substitute for the other. Evidence of scan execution and remediation shall be maintained as a PCI compliance artifact.

## 8. Performance And Scale Requirements

### NFR-01 Throughput

- Platform shall support peak 50,000 RPS aggregate class.
- Platform shall tolerate short bursts above baseline without data loss.

### NFR-02 Latency (Service Side)

- Tokenize: p95 <= 30ms, p99 <= 60ms.
- De-tokenize: p95 <= 35ms, p99 <= 70ms.
- Revoke: p95 <= 50ms, p99 <= 120ms.

### NFR-03 Error Rate

- Under normal peak load, request failure rate shall remain below 1%.

### NFR-04 Concurrency And Fairness

- Platform shall support high concurrent client connections.
- Per-tenant and per-domain rate controls shall prevent noisy-neighbor impact.

## 9. Reliability Requirements

### RR-01 Availability

- Tokenize and detokenize APIs shall target 99.95% monthly availability.

### RR-02 Graceful Degradation

- Cache failures shall degrade to persistent store path.
- Authorization failures shall never degrade to allow-path.

### RR-03 Recovery

- RTO and RPO targets shall be defined and validated via drills.

## 10. Operational Requirements

### OR-01 Observability

- Metrics, logs, and traces shall be correlated by request id.
- Dashboards shall provide endpoint, tenant, and dependency health views.

### OR-02 Alerting

- Alerts shall cover:
  - SLO burn rate
  - latency and error spikes
  - authz anomalies (including detokenize denied spikes and scope mismatch patterns)
  - key access anomalies (unexpected KMS key usage, out-of-context service account activity)
  - dependency saturation
  - PAN data discovery findings outside vault boundary (per SR-08)

### OR-03 Release Safety

- Canary rollout with progressive traffic shifts shall be required.
- Automated rollback triggers shall be configured for latency/error/security violations.

### OR-04 Incident Readiness

- Runbooks shall exist for latency, authorization, and crypto incidents.
- Scheduled game days shall validate response procedures.

## 11. API Contract Requirements

- APIs shall be versioned under `/v1`.
- Request/response schemas shall be formally published in OpenAPI.
- Error model shall be uniform with machine-readable error codes.

## 12. Test And Validation Requirements

### TV-01 Functional Validation

- Unit and integration tests shall cover:
  - reusable vs one-time behavior
  - scope_qualifiers isolation: tokens issued under different qualifier values for the same PAN shall not be retrievable via lookups with different qualifier values
  - idempotency behavior (tokenize) and deduplication window (detokenize)
  - revocation effects by token and by fingerprint selector with and without scope_qualifiers filter
  - authorization denial paths (missing domain permission, missing full-pan scope, invalid reason_code)
  - masked vs full PAN response gating by caller scope

### TV-02 Performance Validation

- Staged load tests at 10k, 25k, and 50k profiles shall be executed.
- Validation shall include mixed workload proportions for tokenize/detokenize/revoke.

### TV-03 Security Validation

- Threat model (STRIDE or equivalent) shall be completed and tracked.
- Authz bypass and key misuse simulations shall be executed.
- Security testing shall include an **OWASP Top 10** threat review per PCI DSS 6.2.4. Specific focus areas for this service:
  - **Injection**: Verify that all Spanner query construction, Redis key lookups, and structured log field population are protected against SQL/NoSQL injection via parameterized queries and strict input validation. No user-supplied or caller-supplied string shall be concatenated directly into a query or key.
  - **Insecure Deserialization**: Verify that all JSON payloads at tokenize, detokenize, and revoke endpoints are deserialized through schema-validated parsers with strict type enforcement. Malformed, oversized, or deeply nested payloads shall be rejected at the gateway layer before reaching service logic.
  - **Broken Access Control**: Verify that scope_qualifiers and domain values in requests cannot be used to escalate privilege or access tokens belonging to other tenants.
  - **Security Misconfiguration**: Verify that no debug endpoints, verbose error responses containing stack traces, or unauthenticated health endpoints expose internal state.
- OWASP Top 10 test coverage shall be documented as a test evidence artifact accompanying each production release candidate.

## 13. Compliance And Governance Requirements

- Controls shall map to PCI DSS requirements and internal policies.
- Evidence repository shall be maintained per release.
- Change approvals and exception handling shall be auditable.

### CR-01 Executive Accountability (PCI DSS 4.0 Req 12.4.1)

- Executive management (CTO or designated VP-level equivalent) shall be formally accountable for maintaining the organization's PCI DSS compliance program. This accountability shall be documented in a charter or policy statement signed by the responsible executive.
- Executive management shall review the organization's PCI compliance posture, open control gaps, remediation status, and risk acceptance decisions at minimum **quarterly**.
- Quarterly review records, including attendees, agenda items, findings reviewed, and decisions made, shall be maintained as compliance evidence in the evidence repository.
- This requirement applies to the tokenization platform as a service provider; compliance cannot be delegated solely to engineering or security teams.

### CR-02 Service Provider Customer Responsibility Acknowledgment (PCI DSS 4.0 Req 12.9.1)

- The organization operates as a PCI DSS service provider storing, processing, or transmitting cardholder data on behalf of its tenants. As such, the organization shall provide each tenant with a written **Shared Responsibility Agreement** at onboarding.
- The Shared Responsibility Agreement shall specify at minimum:
  - The security controls the tokenization platform is responsible for (e.g., vault encryption, key management, access control, audit logging).
  - The controls that remain the tenant's responsibility (e.g., caller identity management, TLS termination on the tenant side, PAN handling prior to tokenize call submission).
  - The cardholder data the platform stores on the tenant's behalf and the protections applied to it.
  - The tenant's right to audit the platform's compliance posture via QSA attestation or SOC 2 report review.
- The Shared Responsibility Agreement shall be reviewed and re-executed **annually** and upon any material change to the platform's security architecture or data handling practices.
- A template Shared Responsibility Agreement shall be reviewed by legal and compliance stakeholders prior to first customer use. Evidence of executed agreements shall be maintained in the compliance evidence repository.

### CR-03 Customized Approach Framework (PCI DSS 4.0)

- Where a specific PCI DSS prescriptive implementation requirement conflicts with the platform's tokenization architecture or technical design, a **Customized Approach** may be proposed as an alternative. A Customized Approach is distinct from a Compensating Control (used when a requirement cannot be met due to legitimate technical or business constraints).
- Any Customized Approach shall:
  - Identify the PCI DSS requirement being addressed and its stated Security Objective.
  - Define the alternative control and provide documented evidence that it achieves an equivalent or greater level of protection than the prescriptive requirement.
  - Be reviewed and formally accepted by the Qualified Security Assessor (QSA) **prior to implementation**.
  - Be tracked in the compliance evidence repository with QSA acceptance artifacts, implementation date, and scheduled review date.
- All active Customized Approaches shall be reviewed at minimum annually and upon any material change to the underlying control.
- The existing compensating control documentation for serverless CDE network controls (SR-02a) shall be assessed to determine whether it should be reclassified as a Customized Approach under PCI DSS 4.0 framing, with QSA guidance.

## 14. Acceptance Criteria

Service is accepted for production when all are true:
- All functional requirements FR-01 to FR-06 and FR-02a are implemented and tested.
- PCI/security requirements SR-01 to SR-08 are implemented with evidence.
- Network/CDE controls (SR-02a) are satisfied either by compensating controls (documented and accepted) or by deployment on a platform with full VPC controls.
- Load tests demonstrate compliance with NFR targets in staging.
- Alerting, runbooks, and rollout controls are validated through drills.
- Compliance and security stakeholders sign off release readiness.
- **SR-01**: Field-level (application-layer) AES-256 encryption is implemented and verified; two distinct Cloud KMS key classes (PEK and PFK) are provisioned with HSM backing and correct IAM scope separation.
- **SR-01 / DR-01**: PAN fingerprint implementation uses HMAC-SHA-256 with a Cloud KMS-managed PFK; no unkeyed hash of PAN is present anywhere in the system.
- **SR-02**: Workload Identity is confirmed as the runtime credential mechanism; no hard-coded credentials exist in source code, CI/CD configs, or container images (validated by SAST scan).
- **SR-03**: Key/data separation of duties is confirmed via IAM policy snapshot; key custodian identity holds no data-plane access and vice versa.
- **SR-07**: MFA enforcement is active at the identity provider level for all CDE-accessible human identities; validated via IdP policy configuration export.
- **SR-08**: Baseline PAN data discovery scan is completed across all CDE-adjacent storage targets with no confirmed out-of-vault findings.
- **TV-03**: OWASP Top 10 test evidence bundle (injection, deserialization, access control, misconfiguration) is produced and attached to the release candidate.
- **CR-01**: Executive compliance review process is established with first quarterly review completed and documented.
- **CR-02**: Shared Responsibility Agreement template is finalized, reviewed by legal, and executed with at least one tenant prior to production launch.
- **CR-03**: Any active Customized Approaches are documented with QSA acceptance artifacts prior to launch.

## 15. Risks And Assumptions

- Assumes trusted internal network controls and service identity issuance are in place.
- 50k TPS achievement depends on production-grade scaling and distributed load generation.
- Single-region design minimizes complexity but increases regional dependency risk.
- Use of serverless (Cloud Run) for CDE may require explicit compensating controls and QSA/SAQ acceptance; GKE or Compute Engine with VPC controls may be preferred for production PCI scope.

## 16. References

- [Tokenizing sensitive cardholder data for PCI DSS \| Google Cloud Architecture Center](https://docs.cloud.google.com/architecture/tokenizing-sensitive-cardholder-data-for-pci-dss) — GCP sample deployment using Cloud Run, Cloud KMS, and IAM; production considerations for network controls and dependency management.
- [PCI Data Security Standard (DSS)](https://www.pcisecuritystandards.org/document_library) — Baseline compliance framework.
- [PCI DSS Tokenization Guidelines Information Supplement](https://www.pcisecuritystandards.org/documents/Tokenization_Guidelines_Info_Supplement_v2_0.pdf) — Token properties and vault requirements.
