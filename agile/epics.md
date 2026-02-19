# Epics

| ID | Epic | Requirements Coverage | Priority |
| --- | --- | --- | --- |
| E-01 | Token Vault — Core Data Model | DR-01, FR-02, FR-02a, SR-01 | P0 |
| E-02 | Tokenize API | FR-01, FR-05 | P0 |
| E-03 | De-Tokenize API (Least Privilege) | FR-03, FR-06, SR-02 | P0 |
| E-04 | Token Revocation API | FR-04, FR-06 | P0 |
| E-05 | Cryptographic Key Management (Institution-Scoped) | SR-01, SR-03 | P0 |
| E-06 | Authorization, Identity, And Delegation | SR-02, SR-02a | P0 |
| E-07 | Audit Logging And Compliance Evidence | FR-06, SR-04, SR-05 | P0 |
| E-08 | Multi-Token Per PAN — Scope Qualifiers | FR-01, FR-02, FR-03, FR-04, DR-01 | P0 |
| E-09 | MFA And Human CDE Access Controls | SR-07 | P1 |
| E-10 | Automated PAN Data Discovery | SR-08 | P1 |
| E-11 | Performance, Scale, And Reliability | NFR-01–04, RR-01–03 | P1 |
| E-12 | Observability, Alerting, And Release Safety | OR-01–04 | P1 |
| E-13 | Security Validation And OWASP Testing | TV-03, SR-05 | P1 |
| E-14 | Service Provider Governance | CR-01, CR-02, CR-03 | P2 |
| E-15 | Multi-Region Active-Active (Phase 2) | Future Phase | P3 |
| E-16 | Institution Onboarding And Key Provisioning | SR-01, SR-02, DR-01, §15 Risks | P0 |
| E-17 | Compute Layer Migration Path (Cloud Run → GKE) | SR-02a, §10 Migration Path | P1 |

## Epic Descriptions

**E-01 Token Vault — Core Data Model**
Build the Spanner vault schema partitioned by `institution_id`, with institution-scoped HMAC fingerprinting, AES-256-GCM field encryption, and indexes supporting reusable token lookup and revocation at 50k RPS.

**E-02 Tokenize API**
Implement `POST /v1/tokenize` supporting REUSABLE and ONE_TIME modes with delegation authorization check, institution-keyed fingerprint computation, and idempotency scoped to `(caller_identity, institution_id, idempotency_key)`.

**E-03 De-Tokenize API (Least Privilege)**
Implement `POST /v1/detokenize` with required `reason_code`, masked PAN default, full-PAN IAM scope gate, delegation authorization check, and fail-closed behavior on any policy error.

**E-04 Token Revocation API**
Implement `POST /v1/tokens/revoke` with token-level and fingerprint-selector revocation, optional `scope_qualifiers` filter for targeted revocation (e.g., decommission one JH application's tokens for a card), and Redis cache eviction.

**E-05 Cryptographic Key Management (Institution-Scoped)**
Provision and manage institution-scoped PFKs and PEKs (both per-institution) in Cloud KMS with HSM backing, 90-day rotation, and key/data separation of duties enforced at IAM. Each institution gets exactly one PFK (HMAC fingerprinting) and one PEK (envelope encryption) provisioned at onboarding.

**E-06 Authorization, Identity, And Delegation**
Build the Delegation Grant Table mapping `(caller_app_identity, institution_id)` to permitted operations/domains/purposes. Enforce Workload Identity for all callers. Implement grant table change approval workflow (dual approval: platform + product security owner).

**E-07 Audit Logging And Compliance Evidence**
Implement immutable audit event stream to Pub/Sub with `caller_application` and `institution_id` as mandatory indexed fields. Enforce log redaction. Support per-institution audit queries in BigQuery for compliance reporting.

**E-08 Multi-Token Per PAN — Scope Qualifiers**
Implement `scope_qualifiers` key-value map for token differentiation by JH application (`{"application": "banno-mobile", "channel": "web"}`), with canonical sort, index support, and scope-filtered revocation.

**E-09 MFA And Human CDE Access Controls**
Enforce MFA at the Google Workspace IdP level for all Jack Henry personnel with CDE resource access. Validate break-glass procedures include MFA. Document minimum 12-character password policy.

**E-10 Automated PAN Data Discovery**
Configure automated scanning across Cloud Logging, BigQuery audit tables, Spanner exports, and CI/CD artifacts for out-of-vault PAN presence. Alert on findings. Run at minimum annually and after infrastructure changes.

**E-11 Performance, Scale, And Reliability**
Validate 50k RPS with p95/p99 SLOs under realistic Jack Henry mixed-application load. Implement per-institution and per-caller-application rate limiting. Validate Redis hit rate ≥ 85% with institution-keyed cache.

**E-12 Observability, Alerting, And Release Safety**
Implement per-institution and per-caller-application dashboards. Alert on delegation anomalies, per-institution detokenize spikes, key access anomalies. Canary deployment with per-institution traffic analysis.

**E-13 Security Validation And OWASP Testing**
Complete STRIDE threat model covering delegation trust risk. OWASP Top 10 test coverage (injection, deserialization, broken access control, misconfiguration). Authenticated internal vulnerability scans quarterly.

**E-14 Service Provider Governance**
Establish executive PCI compliance review cadence. Finalize and execute institution Shared Responsibility Agreement template with Jack Henry Legal. Document Customized Approach framework for any controls deviating from PCI DSS prescriptive requirements.

**E-15 Multi-Region Active-Active (Phase 2)**
Design and implement active-active deployment across two or more U.S. regions with global Spanner consistency, cross-region KMS key access, and sub-5-minute RPO. Requires separate architecture phase.

**E-16 Institution Onboarding And Key Provisioning**
Build the institution onboarding pipeline: provision institution-scoped PFK and PEK in Cloud KMS, register institution in the Institution Registry (Spanner config table), configure delegation grants for authorized JH caller applications, and validate end-to-end tokenize/detokenize flow before production enablement. Both keys are required; no institution may be marked active without both.

**E-17 Compute Layer Migration Path (Cloud Run → GKE)**
Deploy the tokenization service as a Docker container on Cloud Run (Phase 1) for speed, with VPC connector, min-instance configuration, and a QSA-accepted Customized Approach package for PCI DSS 1.2/1.3 network segmentation. When migration criteria are met, migrate the same Docker image to GKE private cluster (Phase 2), retiring the Customized Approach. Application code, API contract, Workload Identity bindings, KMS references, Spanner schema, and Redis configuration are unchanged between phases — only IaC, networking, and autoscaling configuration change.
