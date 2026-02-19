# User Stories

Stories follow: **As a [actor], I want [capability], so that [outcome].**
Acceptance criteria are prefixed with `AC:`.

---

## E-01 — Token Vault: Core Data Model

### S-01-01 Card Record Storage
As the tokenization service, I want to store AES-256-GCM encrypted PAN payloads in Spanner vault tables, so that raw PAN is never present in plaintext in any storage system.
- AC: PAN ciphertext written using PEK via Cloud KMS envelope encryption before insert.
- AC: No plaintext PAN present in any Spanner table, index, or log.
- AC: Integration test verifies encrypted bytes stored; decryption via KMS returns original PAN.

### S-01-02 Token Record Storage With Scope Qualifiers
As the tokenization service, I want to store token records including domain, scope_qualifiers_canonical, purpose, token_mode, state, and a card reference, so that multi-dimensional token uniqueness is enforced in the vault.
- AC: Token record schema includes all required fields including `scope_qualifiers_canonical` (sorted key-value string).
- AC: Unique constraint on `(tenant_id, domain, scope_qualifiers_canonical, purpose, pan_fingerprint)` for REUSABLE tokens.
- AC: Token values are globally unique across all tenants.

### S-01-03 HMAC PAN Fingerprint
As the tokenization service, I want to compute PAN fingerprints using HMAC-SHA-256 with the tenant-scoped PFK, so that indexed lookups are possible without storing raw PAN and brute-force correlation is infeasible.
- AC: Fingerprint computed via Cloud KMS MAC sign operation using PFK.
- AC: No unkeyed SHA-256 or MD5 of PAN exists anywhere in the codebase (SAST validated).
- AC: Same PAN for same tenant always produces same fingerprint (deterministic).
- AC: Different tenants produce different fingerprints for the same PAN (tenant-scoped key).

### S-01-04 Vault Index Design
As an SRE, I want Spanner indexes to support reusable token lookup and fingerprint-based revocation at 50k RPS, so that read latency targets are met without full table scans.
- AC: Primary index: `(tenant_id, domain, scope_qualifiers_canonical, purpose, pan_fingerprint, state)`.
- AC: Secondary index: `(tenant_id, domain, pan_fingerprint, state)` for revocation scans.
- AC: Load test at 50k RPS shows Spanner CPU <= 65% steady-state.

---

## E-02 — Tokenize API

### S-02-01 POST /v1/tokenize — REUSABLE Mode
As a calling service, I want to submit a PAN and receive a stable reusable token scoped to my domain, scope_qualifiers, and purpose, so that the same card is consistently identified without re-tokenizing on each request.
- AC: Caller identity resolved from Workload Identity token; no `tenant_id` accepted in body.
- AC: REUSABLE mode returns existing ACTIVE token if one exists for the scope.
- AC: `reused_existing: true` in response when returning an existing token.
- AC: New token minted if no active token exists for the scope.

### S-02-02 POST /v1/tokenize — ONE_TIME Mode
As a calling service, I want to always receive a fresh single-use token, so that one-time payment flows never reuse a token.
- AC: ONE_TIME mode always mints a new token regardless of existing tokens for the scope.
- AC: One-time tokens are non-reusable after first detokenize or after TTL expiry.

### S-02-03 Tokenize Idempotency
As a calling service, I want duplicate tokenize requests with the same idempotency key to return the original result, so that network retries do not mint duplicate tokens.
- AC: Idempotency scoped to `(caller_identity, idempotency_key)`.
- AC: Replay of same key within retention window returns original token, same state.
- AC: Replay with same key but different PAN returns `409 CONFLICT`.

### S-02-04 Scope Qualifier Pass-Through
As a calling service, I want to pass optional `scope_qualifiers` (e.g., `merchant_id`, `application_id`) in the tokenize request, so that tokens are scoped to specific merchants or applications within a domain.
- AC: `scope_qualifiers` is optional; absence is equivalent to an empty map.
- AC: Tokens with different `scope_qualifiers` values for the same PAN are distinct records.
- AC: `scope_qualifiers` keys and values are validated as non-PAN strings; PAN-like values are rejected with `400 INVALID_REQUEST`.

---

## E-03 — De-Tokenize API (Least Privilege)

### S-03-01 POST /v1/detokenize — Masked PAN Default
As a calling service with detokenize permission, I want to resolve a token to a masked PAN by default, so that full PAN exposure is minimized for most operational use cases.
- AC: Default response returns masked PAN (`411111******1111`) unless `return_type=FULL_PAN` is requested.
- AC: Masked PAN format preserves first 6 and last 4 digits.

### S-03-02 Full PAN Gate By IAM Scope
As the platform, I want to restrict full PAN return to callers with an explicit `full-pan` IAM scope, so that only authorized systems can retrieve cardholder data in cleartext.
- AC: `format_options.return_type=FULL_PAN` without `full-pan` scope returns `403 FORBIDDEN`.
- AC: Full PAN return is audited with `return_type=FULL` in the audit event.
- AC: Masked PAN return is audited with `return_type=MASKED`.

### S-03-03 Reason Code Enforcement
As a compliance officer, I want every detokenize call to require a structured reason code, so that all PAN access is auditable with business intent per PCI DSS Req 10.2.1.2.
- AC: `request_context.reason_code` is required; requests without it return `400 INVALID_REQUEST`.
- AC: Only defined enum values are accepted; free-text is rejected.
- AC: Reason code is written to the audit event.

### S-03-04 Scope Mismatch Non-Disclosure
As the platform, I want a token lookup that fails due to scope mismatch to return `404 TOKEN_NOT_FOUND`, so that callers cannot discover whether a token exists under a different scope.
- AC: `404 TOKEN_NOT_FOUND` returned for: token not found, token revoked, or scope mismatch.
- AC: Error response body does not distinguish between the three cases.

### S-03-05 Fail-Closed Authorization
As a security engineer, I want any policy service error or authorization uncertainty to return a denial, so that PAN is never released on a degraded authorization path.
- AC: Policy service unavailability returns `503` or `403` — never a successful PAN response.
- AC: Chaos test: kill auth policy service mid-request, verify no PAN released.

---

## E-04 — Token Revocation API

### S-04-01 POST /v1/tokens/revoke — By Token
As a calling service, I want to revoke a specific token by value, so that a compromised or expired token is immediately invalidated.
- AC: Token state set to REVOKED in Spanner.
- AC: Token evicted from Redis cache.
- AC: Subsequent detokenize of revoked token returns `404 TOKEN_NOT_FOUND`.
- AC: Audit event published with `selector_type=TOKEN` and `reason`.

### S-04-02 POST /v1/tokens/revoke — By Fingerprint Selector
As a risk operations engineer, I want to revoke all tokens for a PAN within a domain using a fingerprint selector, so that a compromised card can be fully invalidated without enumerating individual tokens.
- AC: Revokes all ACTIVE tokens matching `(tenant, domain, pan_fingerprint)`.
- AC: Optional `scope_qualifiers` filter restricts revocation to matching qualifier values only.
- AC: `revoked_count` in response reflects actual tokens revoked.
- AC: All revoked tokens evicted from Redis.

### S-04-03 Reason Code On Revoke
As the platform, I want revocation requests to require a reason code, so that all revocation events are auditable with a business rationale.
- AC: Missing `reason` returns `400 INVALID_REQUEST`.
- AC: Reason codes include: `merchant_request`, `fraud_signal`, `card_compromised`, `merchant_offboarded`, `application_decommissioned`, `compliance_action`.

---

## E-05 — Cryptographic Key Management

### S-05-01 Two KMS Key Classes — PEK And PFK
As a security engineer, I want separate Cloud KMS key rings for PAN encryption (PEK) and PAN fingerprint HMAC (PFK), so that a compromise of one key class does not expose the other.
- AC: PEK key ring provisioned with `AES-256` symmetric key, HSM-backed.
- AC: PFK key ring provisioned with HMAC key (Cloud KMS MAC), HSM-backed.
- AC: IAM: PEK grants `roles/cloudkms.cryptoKeyEncrypterDecrypter` to service account only.
- AC: IAM: PFK grants MAC sign/verify to service account only.
- AC: Neither key ring grants admin roles to the runtime service account.

### S-05-02 Key Rotation — 90-Day Cadence
As a compliance engineer, I want automatic key rotation configured at 90 days for both PEK and PFK, so that key lifecycle meets PCI DSS 3.6 requirements.
- AC: Cloud KMS rotation period set to 90 days on both key rings.
- AC: Rotation events logged to Cloud Audit Logs.
- AC: After rotation, new encryption uses new key version; old ciphertext remains decryptable via prior key versions.

### S-05-03 Key/Data Separation Of Duties
As a security engineer, I want IAM roles for key administration to be assigned to a separate key custodian identity that has no data-plane access, so that no single principal can both manage keys and access encrypted data.
- AC: Key custodian service account: holds `roles/cloudkms.admin`; does not hold Spanner roles.
- AC: DB admin service account: holds Spanner data-plane roles; does not hold KMS admin roles.
- AC: Runtime service account: holds only KMS encrypter/decrypter/MAC sign/verify; no admin roles.
- AC: IAM policy snapshot reviewed quarterly; evidence artifact produced.

---

## E-06 — Authorization And Identity

### S-06-01 Workload Identity For Runtime Auth
As the platform, I want the tokenization service to authenticate to GCP using Workload Identity, so that no long-lived static credentials are used in production.
- AC: Service deployed with Workload Identity binding to a dedicated service account.
- AC: No service account key files present in container image, source code, or CI/CD config (SAST validated).
- AC: Workload Identity token used for all KMS, Spanner, Redis, and Pub/Sub calls.

### S-06-02 Caller Identity Derived Server-Side
As the platform, I want the caller's tenant to be resolved from their authenticated Workload Identity token, so that callers cannot self-assert a `tenant_id` to access other tenants' data.
- AC: `tenant_id` is not accepted in any request body.
- AC: Tenant resolved from the authenticated identity and used for all vault lookups.
- AC: Cross-tenant access attempt returns `403 FORBIDDEN`.

### S-06-03 Domain And Scope Authorization Policy
As a security engineer, I want authorization rules to enforce permitted `(caller_identity, domain, scope_qualifiers, purpose)` combinations, so that a caller authorized for one domain cannot access tokens in another.
- AC: Policy rules are configurable per caller identity.
- AC: Unauthorized domain access returns `403 FORBIDDEN`.
- AC: Authz policy tests cover: valid combo, wrong domain, wrong purpose, wrong scope_qualifier value.

---

## E-07 — Audit Logging And Compliance Evidence

### S-07-01 Immutable Audit Event Stream
As a compliance engineer, I want every tokenize, detokenize, revoke, and denied request to publish an immutable audit event to Pub/Sub, so that all CHD access is recorded for PCI DSS Req 10 compliance.
- AC: Audit event fields: request_id, caller_identity, action, outcome, timestamp, domain, scope_qualifiers, token_purpose.
- AC: Detokenize events additionally include: reason_code, operator_id (unverified flag), return_type.
- AC: Revoke events additionally include: selector_type, reason, revoked_count.
- AC: No PAN, CVV, key material, or full token value present in any audit event.

### S-07-02 Log Redaction Middleware
As a security engineer, I want structured logging middleware to enforce redaction of PAN, CVV, and key material from all log outputs, so that sensitive data cannot leak through application logs.
- AC: Redaction unit tests cover: PAN in response body, PAN in error messages, PAN in trace labels.
- AC: Sample sanitized logs produced as evidence artifact.
- AC: PAN discovery scanner (E-10) validates no PAN present in Cloud Logging outputs.

---

## E-08 — Multi-Token Per PAN: Scope Qualifiers

### S-08-01 Scope Qualifier Key-Value Map Design
As a platform architect, I want `scope_qualifiers` to be a flexible key-value map rather than fixed fields, so that new scoping dimensions (merchant, application, channel, device) can be added without schema changes.
- AC: `scope_qualifiers` accepts any string key-value pairs.
- AC: Canonicalization: keys sorted lexicographically, values concatenated to form a stable lookup string.
- AC: Key/value max lengths enforced (e.g., 64 chars per key, 256 chars per value).

### S-08-02 Merchant And Application Scoped Token Issuance
As a merchant integration, I want to request tokens scoped to my merchant ID and application ID, so that I receive distinct tokens from other merchants for the same card.
- AC: `scope_qualifiers: {"merchant_id": "m_99821", "application_id": "app-mobile-v2"}` produces a token unique to that merchant/app scope.
- AC: Same PAN, same domain, same purpose — but different merchant_id — produces a different REUSABLE token.
- AC: Integration test validates token isolation across merchant scope qualifier values.

### S-08-03 Scope-Filtered Revocation
As a risk operations engineer, I want to revoke all tokens for a specific merchant when they are offboarded, without revoking tokens for other merchants on the same card.
- AC: Revoke by fingerprint with `scope_qualifiers: {"merchant_id": "m_99821"}` revokes only tokens matching that qualifier.
- AC: Tokens for `{"merchant_id": "m_00001"}` on the same card are unaffected.
- AC: Integration test validates selective revocation.

---

## E-09 — MFA And Human CDE Access Controls

### S-09-01 IdP-Level MFA Enforcement
As a security engineer, I want MFA enforced at the Google Workspace IdP level for all CDE-scoped resource access, so that individual users cannot bypass MFA through settings changes.
- AC: Google Workspace 2-Step Verification enforcement policy applied to the CDE project group.
- AC: No user or admin can disable MFA for their own account while in CDE group.
- AC: IdP enforcement config exported as evidence artifact.

### S-09-02 MFA In Break-Glass Procedure
As a security engineer, I want the break-glass emergency access procedure to require MFA, so that emergency access paths cannot be used to bypass MFA controls.
- AC: Break-glass runbook documents MFA as a required step.
- AC: If MFA is unavailable during break-glass, procedure escalates to dual-control manual process.
- AC: Break-glass access events are audited in Cloud Audit Logs.

---

## E-10 — Automated PAN Data Discovery

### S-10-01 Annual PAN Discovery Scan
As a compliance engineer, I want automated tooling to scan all CDE-adjacent storage locations for out-of-vault PAN presence, so that any accidental PAN leakage is detected per PCI DSS 12.5.2.
- AC: Scan targets: Cloud Logging buckets, BigQuery audit tables, Error Reporting, Spanner exports, Redis snapshots, CI/CD artifact storage.
- AC: Scan executed at minimum annually and after significant infrastructure changes.
- AC: Scan report produced within 30 days; findings trigger incident response per OR-04.

### S-10-02 Alerting On Discovery Findings
As an SRE, I want Cloud Monitoring alerts to trigger on PAN discovery findings outside the vault boundary, so that any leakage is actioned without waiting for periodic reviews.
- AC: Alert configured in Cloud Monitoring tied to PAN discovery tool output.
- AC: Alert routes to on-call channel and creates an incident ticket.

---

## E-11 — Performance, Scale, And Reliability

### S-11-01 50k RPS Sustained Throughput
As an SRE, I want the platform to sustain 50,000 RPS aggregate with p95 tokenize latency <= 30ms and p95 detokenize latency <= 35ms, so that peak payment volume is handled without degradation.
- AC: Load test stages: 10k, 25k, 50k RPS sustained for 2 minutes each.
- AC: All NFR-02 latency SLOs met at 50k RPS.
- AC: Error rate < 1% under normal peak load.

### S-11-02 Redis Cache Hit Rate
As an SRE, I want Redis cache hit rate >= 85% for REUSABLE token hot reads, so that Spanner load remains within capacity limits.
- AC: Cache keyed on `(tenant_id, domain, scope_qualifiers_canonical, purpose, pan_fingerprint)`.
- AC: Cache hit rate monitored and dashboarded.
- AC: Cache failure degrades to Spanner read path without error to caller.

### S-11-03 Graceful Degradation
As an SRE, I want authorization failures to always deny access, even when the policy service is degraded, so that PAN is never released on a failed auth path.
- AC: Chaos test: auth policy service killed mid-request — all in-flight detokenize calls return error.
- AC: No PAN released during policy service outage.

---

## E-12 — Observability, Alerting, And Release Safety

### S-12-01 Request-Correlated Traces
As an SRE, I want all logs, metrics, and traces correlated by `request_id`, so that end-to-end request debugging is possible across gateway, API, Spanner, Redis, and audit sink.
- AC: `request_id` propagated in all log entries, trace spans, and audit events.
- AC: Cloud Trace shows full request path from gateway to audit event.

### S-12-02 Canary Rollout With Auto-Rollback
As an SRE, I want deployments to use canary traffic shifting with automated rollback on latency, error, or security violations, so that bad releases are caught before reaching full traffic.
- AC: Canary at 5% traffic for minimum 10 minutes before promotion.
- AC: Automated rollback triggers on: p99 latency breach, error rate > 1%, authz anomaly spike.

---

## E-13 — Security Validation And OWASP Testing

### S-13-01 OWASP Top 10 Test Coverage
As a security engineer, I want OWASP Top 10 tests covering injection, insecure deserialization, broken access control, and security misconfiguration for each release, so that common vulnerability classes are validated per PCI DSS 6.2.4.
- AC: Injection test: attempt SQL/NoSQL injection via `domain`, `scope_qualifiers`, and `token` fields.
- AC: Deserialization test: submit malformed, oversized, and deeply nested JSON payloads.
- AC: Access control test: attempt cross-tenant and cross-scope token access.
- AC: Misconfiguration test: verify no debug endpoints, stack traces in errors, or unauthenticated health endpoints expose internal state.
- AC: OWASP evidence bundle attached to each release candidate.

### S-13-02 Authenticated Internal Vulnerability Scans
As a compliance engineer, I want quarterly authenticated internal vulnerability scans, so that OS-level and application-layer vulnerabilities are detected per PCI DSS 11.3.1.2.
- AC: Scans conducted with credentials sufficient for OS patch assessment.
- AC: Evidence distinguishes scan tier (a) unauthenticated external, (b) unauthenticated internal, (c) authenticated internal.
- AC: Remediation records maintained in compliance evidence repository.

---

## E-14 — Service Provider Governance

### S-14-01 Executive Compliance Review
As the compliance team, I want a documented quarterly executive review of PCI compliance posture, so that executive accountability is established per PCI DSS 12.4.1.
- AC: Review calendar established with named executive owner.
- AC: First quarterly review completed before production launch.
- AC: Review minutes including attendees, findings, and decisions maintained as evidence.

### S-14-02 Shared Responsibility Agreement
As a tenant, I want a written Shared Responsibility Agreement at onboarding that defines what the platform secures vs. what I am responsible for, so that compliance obligations are clear per PCI DSS 12.9.1.
- AC: Agreement template reviewed by legal and compliance before first use.
- AC: Agreement executed with each tenant at onboarding.
- AC: Agreement re-executed annually and on material platform changes.

### S-14-03 Customized Approach Documentation
As a compliance engineer, I want any Customized Approach to a PCI DSS requirement documented with the Security Objective and QSA acceptance prior to implementation, so that alternative controls are formally recognized.
- AC: Active Customized Approaches tracked in compliance evidence repository.
- AC: Each entry includes: PCI DSS requirement, Security Objective, alternative control description, QSA acceptance artifact, implementation date, next review date.
- AC: SR-02a serverless CDE control assessed for reclassification as Customized Approach vs. Compensating Control.

---

## E-15 — Multi-Region Active-Active (Phase 2)

### S-15-01 Global Spanner Vault Replication
As an SRE, I want the token vault to replicate across at least two U.S. regions with strong consistency, so that a regional failure does not result in data loss or incorrect token deduplication.
- AC: Cloud Spanner multi-region configuration with RPO < 5 minutes.
- AC: Reusable token uniqueness maintained globally across regions.

### S-15-02 Cross-Region KMS Key Access
As the tokenization service, I want PEK and PFK Cloud KMS keys accessible from multiple regions, so that tokenize and detokenize operations succeed during a regional KMS control plane event.
- AC: KMS key ring configured for multi-region or dual-region availability.
- AC: Failover test validates KMS access during simulated regional degradation.

### S-15-03 Active-Active Traffic Routing
As an SRE, I want API traffic routed across regions with automatic failover, so that a regional outage does not cause downtime exceeding the 99.95% SLO.
- AC: Global load balancer (Cloud Load Balancing) routes to nearest healthy region.
- AC: Health check failover time <= 30 seconds.
- AC: DR drill validates RPO and RTO targets.
