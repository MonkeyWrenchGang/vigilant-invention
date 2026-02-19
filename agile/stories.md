# User Stories

Stories follow: **As a [actor], I want [capability], so that [outcome].**
Acceptance criteria are prefixed with `AC:`.

---

## E-01 — Token Vault: Core Data Model

### S-01-01 Institution-Partitioned Card Record Storage
As the tokenization service, I want card records stored in Spanner partitioned by `institution_id` with AES-256-GCM encrypted PAN, so that cardholder data is physically isolated per Jack Henry institution client.
- AC: `institution_id` is the leading key component on all Spanner card record rows.
- AC: PAN ciphertext written using institution PEK via Cloud KMS envelope encryption before insert.
- AC: No plaintext PAN present in any Spanner table, index, or log.
- AC: Cross-institution query (attempting to read institution B rows with institution A credentials) returns no results.

### S-01-02 Institution-Partitioned Token Record Storage
As the tokenization service, I want token records to store `institution_id`, `domain`, `scope_qualifiers_canonical`, `purpose`, `token_mode`, `state`, `caller_application`, and a card reference, so that multi-dimensional token uniqueness is enforced per institution.
- AC: Token record schema includes all required fields.
- AC: `institution_id` is the leading Spanner key component.
- AC: Unique constraint on `(institution_id, domain, scope_qualifiers_canonical, purpose, pan_fingerprint)` for REUSABLE tokens.
- AC: Token values are globally unique across all institutions.

### S-01-03 Institution-Scoped HMAC PAN Fingerprint
As the tokenization service, I want PAN fingerprints computed using HMAC-SHA-256 with the institution-scoped PFK, so that the same PAN at two different institutions produces different fingerprints, preventing cross-institution correlation.
- AC: Fingerprint computed via Cloud KMS MAC sign operation using the institution's PFK (resolved from Institution Registry).
- AC: No unkeyed SHA-256 or MD5 of PAN exists anywhere in the codebase (SAST validated).
- AC: Same PAN + same institution always produces same fingerprint (deterministic).
- AC: Same PAN + different institutions produces different fingerprints (institution-scoped key verification test).

### S-01-04 Vault Index Design For Institution-Scoped Lookups
As an SRE, I want Spanner indexes to support reusable token lookup and fingerprint-based revocation at 50k RPS with institution_id as the leading dimension, so that queries are physically scoped to a single institution and do not fan out.
- AC: Primary index: `(institution_id, domain, scope_qualifiers_canonical, purpose, pan_fingerprint, state)`.
- AC: Secondary index: `(institution_id, domain, pan_fingerprint, state)` for revocation scans.
- AC: Load test at 50k RPS across multiple institutions shows Spanner CPU ≤ 65% steady-state.
- AC: No cross-institution index scan possible by design.

---

## E-02 — Tokenize API

### S-02-01 POST /v1/tokenize — REUSABLE Mode With Institution Authorization
As a Banno service, I want to submit a PAN with an `institution_id` and receive a stable reusable token for that institution's card, so that the same card at First National Bank is tracked separately from the same card at a credit union.
- AC: `institution_id` is required; missing or empty returns `400 INVALID_REQUEST`.
- AC: Delegation grant check: Banno service account not authorized for the supplied institution_id returns `403 FORBIDDEN`.
- AC: REUSABLE mode returns existing ACTIVE token if one exists for `(institution_id, domain, scope_qualifiers_canonical, purpose, pan_fingerprint)`.
- AC: `reused_existing: true` in response when returning an existing token.

### S-02-02 POST /v1/tokenize — ONE_TIME Mode
As a SilverLake service, I want to mint a fresh single-use token for each transaction, so that one-time payment flows never reuse a token across institution transactions.
- AC: ONE_TIME mode always mints a new token regardless of existing tokens for the scope.
- AC: `institution_id` validated against delegation grant before minting.

### S-02-03 Tokenize Idempotency Scoped To Caller + Institution
As a Banno service, I want duplicate tokenize requests with the same idempotency key for the same institution to return the original result, so that network retries do not mint duplicate tokens.
- AC: Idempotency scoped to `(caller_identity, institution_id, idempotency_key)`.
- AC: Same idempotency key used by Banno for institution A and institution B are treated as distinct — both succeed independently.
- AC: Replay with same key returns original token.
- AC: Replay with same key but different PAN returns `409 CONFLICT`.

### S-02-04 Scope Qualifier Pass-Through For JH Application Context
As a Banno mobile service, I want to pass `{"application": "banno-mobile", "channel": "web"}` as scope qualifiers, so that tokens issued through the mobile channel are distinct from tokens issued through the web channel for the same institution card.
- AC: `scope_qualifiers` is optional; absence is equivalent to empty map.
- AC: Tokens with different `scope_qualifiers` for the same institution/PAN are distinct vault records.
- AC: PAN-like values in scope_qualifier keys or values are rejected with `400 INVALID_REQUEST`.

---

## E-03 — De-Tokenize API (Least Privilege)

### S-03-01 POST /v1/detokenize — Institution Authorization And Masked Default
As a SilverLake service, I want to resolve a token to a masked PAN for an institution's card, so that the cardholder number is protected by default for most operational use cases.
- AC: `institution_id` is required; delegation grant validated before any vault access.
- AC: Default response returns masked PAN (`411111******1111`).
- AC: Masked PAN format preserves first 6 and last 4 digits.

### S-03-02 Full PAN Gate By IAM Scope
As the platform, I want full PAN return restricted to callers holding an explicit `full-pan` IAM scope, so that only Jack Henry services with a documented business need can retrieve cleartext cardholder data.
- AC: `format_options.return_type=FULL_PAN` without `full-pan` IAM scope returns `403 FORBIDDEN`.
- AC: Both masked and full PAN returns are audited with `return_type` field in the audit event.

### S-03-03 Reason Code Enforcement For Institution Audit Trail
As a compliance engineer, I want every detokenize call to require a structured reason code, so that all PAN access is attributable to a specific institution business context for PCI DSS Req 10.2.1.2 compliance.
- AC: `request_context.reason_code` is required; requests without it return `400 INVALID_REQUEST`.
- AC: Only defined enum values accepted (`PAYMENT_PROCESSING`, `REFUND_PROCESSING`, `FRAUD_INVESTIGATION`, `CHARGEBACK`, `SETTLEMENT`, `COMPLIANCE_REVIEW`).
- AC: Reason code written to audit event alongside `institution_id` and `caller_application`.

### S-03-04 Operator ID — Required For Full-PAN, Optional For Masked-PAN
As a compliance engineer, I want `operator_id` required on all full-PAN detokenize requests and optional on masked-PAN requests, so that every cleartext PAN access has an attributed bank employee or operator identity in the audit record.
- AC: `format_options.return_type=FULL_PAN` without `operator_id` returns `400 INVALID_REQUEST`.
- AC: `operator_id` written to audit event with `"verified": false` flag.
- AC: Audit query by `institution_id` + `operator_id` returns all detokenize events for that operator at that institution.
- AC: Absence of `operator_id` on a masked-PAN request is accepted.

### S-03-05 Institution And Scope Mismatch Non-Disclosure
As the platform, I want token lookups that fail due to wrong `institution_id` or `scope_qualifiers` to return `404 TOKEN_NOT_FOUND`, so that callers cannot discover whether a token exists at a different institution.
- AC: Token not found, token revoked, institution mismatch, and scope mismatch all return `404 TOKEN_NOT_FOUND`.
- AC: Error body does not distinguish between the four cases.

### S-03-06 Fail-Closed Authorization Including Delegation Failures
As a security engineer, I want any delegation grant error, policy service failure, or authorization uncertainty to result in a denial, so that PAN is never released during a degraded authorization state.
- AC: Delegation Grant Table unavailability returns `503` — never a successful PAN response.
- AC: Chaos test: kill Grant Table service mid-request, verify all in-flight detokenize calls return error, no PAN released.

---

## E-04 — Token Revocation API

### S-04-01 POST /v1/tokens/revoke — By Token With Institution Check
As a Banno service, I want to revoke a specific token for an institution's card, so that a compromised or replaced card token is immediately invalidated for that institution.
- AC: `institution_id` required; delegation grant validated before revocation.
- AC: Token state set to REVOKED in Spanner; Redis cache evicted.
- AC: Subsequent detokenize of revoked token returns `404 TOKEN_NOT_FOUND`.
- AC: Audit event published with `caller_application`, `institution_id`, `selector_type=TOKEN`, and `reason`.

### S-04-02 POST /v1/tokens/revoke — By Fingerprint With Scope Filter
As a risk operations engineer at Jack Henry, I want to revoke all Banno-issued tokens for a compromised card at a specific institution, without affecting SilverLake-issued tokens for the same card, so that token revocation is precisely targeted by JH application context.
- AC: Fingerprint revocation with `scope_qualifiers: {"application": "banno-mobile"}` revokes only tokens matching that qualifier.
- AC: Tokens with `scope_qualifiers: {"application": "silverlake-core"}` for the same card/institution are unaffected.
- AC: Omitting `scope_qualifiers` revokes all tokens for the fingerprint within `(institution_id, domain)`.
- AC: `revoked_count` reflects actual tokens revoked.

### S-04-03 Institution-Appropriate Revocation Reason Codes
As the platform, I want revocation reason codes to reflect Jack Henry's institutional context, so that audit records use business-meaningful terminology for bank and credit union operations.
- AC: Accepted codes: `institution_request`, `fraud_signal`, `card_compromised`, `card_reissued`, `application_decommissioned`, `compliance_action`.
- AC: Missing `reason` returns `400 INVALID_REQUEST`.
- AC: Reason code written to audit event.

---

## E-05 — Cryptographic Key Management (Institution-Scoped)

### S-05-01 Institution-Scoped PFK Provisioning
As a security engineer, I want each institution to have its own Cloud KMS HMAC key (PFK), so that PAN fingerprints are institution-unique and cross-institution correlation is structurally impossible.
- AC: One KMS HMAC key per institution; no sharing of PFKs across institutions.
- AC: PFK key ring provisioned HSM-backed with MAC sign/verify permission granted only to tokenization service runtime account.
- AC: Institution Registry records the PFK key name for each institution.
- AC: Test: same PAN at institution A and institution B produces different fingerprints (different PFKs).

### S-05-02 Institution-Scoped PEK Provisioning
As a security engineer, I want each institution to have its own Cloud KMS AES-256-GCM envelope encryption key (PEK), so that a compromised PEK affects only one institution and the encryption isolation posture is QSA-acceptable.
- AC: One KMS PEK per institution; no sharing of PEKs across institutions.
- AC: PEK key ring provisioned HSM-backed with `roles/cloudkms.cryptoKeyEncrypterDecrypter` granted only to tokenization service runtime account.
- AC: Institution Registry records the PEK key name alongside the PFK key name for each institution.
- AC: PEK provisioned as part of institution onboarding pipeline (S-16-01) before any tokenize/detokenize operations for that institution.

### S-05-03 Key Rotation — 90-Day Cadence For All Institution Keys
As a compliance engineer, I want automatic key rotation configured at 90 days for all institution PFKs and PEKs, so that PCI DSS 3.6 key lifecycle requirements are met across all institutions.
- AC: Cloud KMS rotation period set to 90 days on all PFK and PEK key rings.
- AC: Rotation events logged and monitored.
- AC: After rotation, new operations use new key version; prior ciphertext remains decryptable via prior key versions.
- AC: Evidence: rotation logs for each institution's keys available for quarterly audit.

### S-05-04 Key/Data Separation Of Duties
As a security engineer, I want key custodian IAM identity to have no data-plane access to Spanner vault tables, so that no single principal can both manage keys and access encrypted cardholder data.
- AC: Key custodian: holds `roles/cloudkms.admin`; does not hold Spanner or Redis data-plane roles.
- AC: DB admin: holds Spanner roles; does not hold KMS admin roles.
- AC: Runtime service account: holds only KMS encrypter/decrypter and MAC sign/verify; no admin roles.
- AC: Quarterly IAM snapshot review produces evidence artifact per SR-05.

---

## E-06 — Authorization, Identity, And Delegation

### S-06-01 Workload Identity For All JH Caller Applications
As the platform, I want all Banno, SilverLake, Symitar, and ProfitStars services to authenticate via Workload Identity, so that no static credentials are used for any JH application calling the tokenization service.
- AC: Each JH product team has a dedicated service account in the tokenization service's authorized caller list.
- AC: No service account key files present in any JH application's container image, source, or CI/CD (SAST validated per product team).
- AC: Workload Identity tokens validated at the API gateway before reaching service logic.

### S-06-02 Delegation Grant Table — Caller App To Institution Authorization
As the platform, I want a grant table that maps each JH caller application to its authorized institutions, domains, and purposes, so that Banno cannot access SilverLake's institution data and no application can access an institution it has not been explicitly authorized for.
- AC: Grant table stored as platform-managed config; not caller-modifiable.
- AC: Request with `institution_id` not in caller's grant set returns `403 FORBIDDEN`.
- AC: Grant table supports per-domain and per-purpose scoping within an institution grant.
- AC: Integration test: Banno service account attempting to access institution authorized only for SilverLake returns `403`.

### S-06-03 Delegation Grant Change Approval Workflow
As the platform owner, I want grant table changes to require approval from both the platform team and the relevant product team's security owner, so that institution access grants are never unilaterally added or removed.
- AC: Grant change process documented in runbook.
- AC: All grant table changes produce an audit record with approver identities and timestamps.
- AC: Emergency grant revocation (e.g., suspected compromise) can be performed by platform security with single approval and retrospective review.

### S-06-04 Institution Registry Lookup
As the tokenization service, I want to look up an institution's PFK and PEK key references from a central registry at request time, so that the service does not hardcode key names and new institutions can be onboarded without code changes.
- AC: Institution Registry is a **dedicated Spanner config table** within the platform's Spanner instance, with IAM roles separated from vault data tables (registry table has no overlap with vault table grants).
- AC: Registry lookup adds ≤ 2ms to request latency (in-process cached with 60-second TTL; Spanner only hit on cold start or cache miss).
- AC: Unknown `institution_id` (not in registry) returns `403 FORBIDDEN` — same as delegation failure, no disclosure.
- AC: Registry entries cannot be modified by caller applications; only platform onboarding pipeline holds write access.

---

## E-07 — Audit Logging And Compliance Evidence

### S-07-01 Dual-Keyed Audit Events: Caller Application And Institution
As a compliance engineer, I want every audit event to carry both `caller_application` (which JH product) and `institution_id` (which bank/credit union), so that I can produce per-institution compliance reports and per-application anomaly reports independently.
- AC: All audit event schemas include `caller_application` and `institution_id` as mandatory, indexed fields.
- AC: BigQuery audit table is queryable by `institution_id` alone for institution-level compliance reporting.
- AC: BigQuery audit table is queryable by `caller_application` alone for JH product-level anomaly detection.
- AC: No PAN, CVV, key material, or full token value present in any audit event.

### S-07-02 Per-Institution Audit Report Support
As a bank client compliance officer (via Jack Henry account team), I want audit reports filterable by my `institution_id`, so that I can review all tokenization operations performed on behalf of my institution for PCI DSS compliance evidence.
- AC: BigQuery query by `institution_id` returns all tokenize, detokenize, revoke, and denied events for that institution.
- AC: Report output contains: timestamp, caller_application, action, outcome, domain, scope_qualifiers, reason_code (detokenize), operator_id (detokenize).
- AC: Report output never contains PAN, CVV, or cleartext cardholder data.

---

## E-08 — Multi-Token Per PAN: Scope Qualifiers

### S-08-01 JH Application And Channel Scoped Token Issuance
As a Banno mobile service, I want to request tokens with `{"application": "banno-mobile", "channel": "web"}` scope qualifiers, so that a card tokenized through the Banno web channel has a distinct token from the same card tokenized through the Banno iOS app.
- AC: Different `scope_qualifiers` values for the same institution/PAN/domain produce different REUSABLE token records.
- AC: Integration test validates token isolation across `channel` scope qualifier values within Banno's institution context.

### S-08-02 Application Decommission Via Scope-Filtered Revocation
As a platform engineer, I want to revoke all tokens issued under a decommissioned JH application's scope qualifier, so that tokens for a retired product line are invalidated without affecting tokens from other applications for the same institution cards.
- AC: Revoke by fingerprint with `scope_qualifiers: {"application": "legacy-app"}` revokes only tokens matching that qualifier at the specified institution.
- AC: Tokens with `scope_qualifiers: {"application": "banno-mobile"}` for the same institution/card are unaffected.
- AC: Reason code `application_decommissioned` is accepted and written to audit event.

---

## E-09 — MFA And Human CDE Access Controls

### S-09-01 IdP-Level MFA For JH Personnel
As a security engineer, I want MFA enforced at the Google Workspace IdP level for all Jack Henry personnel with CDE resource access, so that no JH developer, SRE, or contractor can access production vault, KMS, or Spanner without a second factor.
- AC: Google Workspace 2-Step Verification enforcement policy applied to the CDE project group.
- AC: No user can disable MFA for their own account while in the CDE access group.
- AC: Validated via IdP policy config export as evidence artifact.

### S-09-02 MFA In Break-Glass With Dual-Control Fallback
As a security engineer, I want break-glass emergency access to require MFA, and if MFA is unavailable, to require dual-control manual procedure, so that no single person can access CDE data in an emergency without oversight.
- AC: Break-glass runbook documents MFA as step 1.
- AC: MFA unavailability escalates to dual-control procedure (two named approvers required).
- AC: All break-glass access events appear in Cloud Audit Logs with accessing identity.

---

## E-10 — Automated PAN Data Discovery

### S-10-01 Annual PAN Discovery Across All CDE-Adjacent Storage
As a compliance engineer, I want automated tooling to scan all CDE-adjacent storage for unprotected PAN presence, so that any accidental leakage across the Jack Henry platform is detected per PCI DSS 12.5.2.
- AC: Scan targets include: Cloud Logging buckets, BigQuery audit tables (including institution-scoped audit records), Cloud Error Reporting, Spanner table exports, Redis snapshots, CI/CD artifact storage.
- AC: Scan executed minimum annually and after any significant infrastructure change.
- AC: Scan report reviewed within 30 days; PAN findings outside vault boundary trigger OR-04 incident procedures.
- AC: Evidence of scan execution and remediation maintained per SR-05.

### S-10-02 PAN Discovery Alert Integration
As an SRE, I want Cloud Monitoring alerts to fire on PAN discovery findings, so that any leakage is actioned without waiting for scheduled review periods.
- AC: Alert configured and routed to on-call channel.
- AC: Alert includes: scan target, finding location, institution_id context if determinable.

---

## E-11 — Performance, Scale, And Reliability

### S-11-01 50k RPS Across Multi-Institution Mixed Load
As an SRE, I want the platform to sustain 50,000 RPS aggregate with SLO targets met under realistic Jack Henry multi-institution, multi-application load profiles, so that peak payment volume across all institutions is handled without degradation.
- AC: Load test simulates traffic from multiple JH caller applications (Banno, SilverLake) across multiple institutions.
- AC: Tokenize p95 ≤ 30ms, p99 ≤ 60ms. Detokenize p95 ≤ 35ms, p99 ≤ 70ms at 50k RPS.
- AC: Error rate < 1%.

### S-11-02 Per-Institution And Per-App Rate Limiting
As an SRE, I want rate limits enforced per institution and per caller application, so that a single large bank client or a single JH product line cannot degrade service for others.
- AC: Per-institution default: 500 RPS, 2,000 RPS burst. Hard ceiling: 10,000 RPS.
- AC: Per-caller-application default: 5,000 RPS. Hard ceiling requires platform team approval.
- AC: Rate limit exceeded returns `429 RATE_LIMITED`.
- AC: Rate limit buckets are institution-isolated — one institution's burst does not consume another's quota.

### S-11-03 Institution-Keyed Redis Cache
As an SRE, I want the Redis hot cache to be keyed by `institution_id` as the leading dimension, so that institution A's cache entries cannot be served to institution B and cache hit rate is measurable per institution.
- AC: Redis cache key format: `{institution_id}:{domain}:{scope_qualifiers_canonical}:{purpose}:{pan_fingerprint}`.
- AC: Cache hit rate ≥ 85% for hot reads measured per institution.
- AC: Cache failure degrades to Spanner read path; no cross-institution data leakage during cache failure.

---

## E-12 — Observability, Alerting, And Release Safety

### S-12-01 Per-Institution And Per-Application Dashboards
As an SRE, I want Cloud Monitoring dashboards that show request volume, latency, and error rates broken down by both `institution_id` and `caller_application`, so that I can distinguish a Banno-specific issue from an institution-specific issue.
- AC: Dashboard dimensions: endpoint, institution_id, caller_application, outcome.
- AC: Delegation grant denial rate visible as a separate metric.

### S-12-02 Delegation Anomaly Alerting
As a security engineer, I want alerts to fire on unusual delegation denial patterns, so that a misconfigured or potentially compromised JH application attempting cross-institution access is detected quickly.
- AC: Alert on: spike in `403 FORBIDDEN` for a given `(caller_application, institution_id)` combination.
- AC: Alert on: any caller application attempting an institution_id it has never previously accessed.
- AC: Alert routes to security on-call channel in addition to SRE on-call.

---

## E-13 — Security Validation And OWASP Testing

### S-13-01 Delegation Bypass And Cross-Institution Attack Testing
As a security engineer, I want threat model and pen test coverage of delegation bypass, cross-institution token access, and institution_id injection, so that the trust model's attack surface is validated before production.
- AC: STRIDE threat model includes delegation trust risk scenario.
- AC: Pen test includes: attempt institution_id substitution in requests; attempt Spanner cross-institution query via crafted token; attempt grant table manipulation.
- AC: All OWASP Top 10 tests updated to use institution-scoped request payloads.

### S-13-02 Authenticated Vulnerability Scans With Institution Context
As a compliance engineer, I want quarterly authenticated internal scans to cover the institution registry and delegation grant table as attack surface, so that configuration vulnerabilities in these critical components are detected.
- AC: Authenticated scans cover: tokenization service OS/app layer, Spanner config, KMS IAM bindings, grant table access controls.
- AC: Evidence distinguishes scan tiers (a), (b), (c) per SR-05.

---

## E-14 — Service Provider Governance

### S-14-01 Executive PCI Compliance Review — Jack Henry Leadership
As the Jack Henry compliance team, I want quarterly executive-level PCI compliance reviews covering the tokenization platform and all institution obligations, so that Jack Henry leadership maintains formal accountability for its service provider obligations.
- AC: First quarterly review completed before production launch.
- AC: Review covers: open control gaps, active institution Shared Responsibility Agreements, SR-08 discovery scan status.

### S-14-02 Institution Shared Responsibility Agreement
As a bank or credit union client, I want a written agreement from Jack Henry specifying which security controls they own for the tokenization service vs. what remains my responsibility, so that my PCI DSS compliance posture is clear.
- AC: Agreement template reviewed by Jack Henry Legal.
- AC: Agreement specifies: Jack Henry owns vault encryption, institution-scoped key management, access control, audit logging; institution owns PAN handling within their own systems.
- AC: Agreement specifies institution's right to review Jack Henry SOC 2 Type II and QSA attestation.
- AC: Agreement executed with each institution at onboarding; re-executed annually.

### S-14-03 Customized Approach Documentation
As a compliance engineer, I want any Customized Approach to PCI DSS requirements documented with QSA acceptance before implementation, so that non-standard controls are formally recognized by our assessor.
- AC: SR-02a serverless CDE network controls assessed for Customized Approach vs. Compensating Control classification.
- AC: Each active Customized Approach tracked with: PCI DSS requirement, Security Objective, alternative control description, QSA acceptance artifact.

---

## E-15 — Multi-Region Active-Active (Phase 2)

### S-15-01 Global Spanner With Institution-Partitioned Replication
As an SRE, I want the institution-partitioned Spanner vault replicated across at least two U.S. regions, so that a regional failure does not cause data loss or cross-institution consistency issues.
- AC: Cloud Spanner multi-region configuration with RPO < 5 minutes.
- AC: Institution-scoped token uniqueness maintained globally across regions.

### S-15-02 Cross-Region Institution PFK Access
As the tokenization service, I want institution PFKs accessible from multiple regions, so that HMAC fingerprint computation succeeds during a regional KMS control plane event without cross-region key sharing violations.
- AC: KMS key rings configured for multi-region or dual-region availability per institution.
- AC: Failover test validates institution PFK access during simulated regional degradation.

---

## E-16 — Institution Onboarding And Key Provisioning

### S-16-01 Institution PFK Provisioning Pipeline
As a platform engineer, I want an automated pipeline to provision a new institution-scoped PFK in Cloud KMS when a new institution is onboarded, so that manual key creation errors are eliminated and the process is auditable.
- AC: Pipeline input: `institution_id`, institution name, key ring config.
- AC: Pipeline output: Cloud KMS HMAC key created, HSM-backed, 90-day rotation configured, IAM grants applied.
- AC: Key name registered in Institution Registry.
- AC: Provisioning event logged as onboarding audit artifact.
- AC: Pipeline is idempotent: re-running for an existing institution_id does not create a duplicate key.

### S-16-02 Institution Registry Entry Creation
As a platform engineer, I want institution onboarding to create an Institution Registry entry mapping `institution_id` to its PFK and PEK key names, so that the tokenization service can resolve key references at runtime without hardcoding.
- AC: Registry entry created only after both PFK (S-16-01) and PEK (S-05-02) have been provisioned; onboarding pipeline shall fail if either key is absent.
- AC: Registry entry includes: `institution_id`, `pfk_key_name`, `pek_key_name`, `status=pending_validation`, `created_at`. Both key name fields are required; entry with a null PEK key name is invalid.
- AC: `status` is only set to `active` after S-16-04 smoke test passes.
- AC: Registry lookup latency ≤ 2ms (in-process cache with 60-second TTL).

### S-16-03 Delegation Grant Configuration For New Institution
As a platform engineer, I want to configure which JH caller applications are authorized to process for a newly onboarded institution and under which domains and purposes, so that the delegation grant table is correctly populated before the institution goes live.
- AC: Grant table entry created specifying: `caller_app_identity`, `institution_id`, allowed `domains`, allowed `purposes`, `full_pan_allowed` flag.
- AC: Grant configuration requires dual approval (platform team + product team security owner).
- AC: Grant table entry visible in audit log.

### S-16-04 End-To-End Onboarding Validation
As a platform engineer, I want an automated smoke test run after institution onboarding completes, validating tokenize → detokenize → revoke for the new institution before it is marked as production-ready, so that misconfigured keys or grants are caught at onboarding, not in production.
- AC: Smoke test: tokenize a test PAN for new institution → detokenize with masked PAN → revoke → confirm revoked token returns 404.
- AC: Smoke test uses a synthetic test PAN, not real cardholder data.
- AC: Smoke test failure blocks institution from being set to `status=active` in the registry.
- AC: Smoke test results logged as onboarding evidence artifact.

---

## E-17 — Compute Layer Migration Path (Cloud Run → GKE)

### S-17-01 Cloud Run Phase 1 Deployment
As a platform engineer, I want to deploy the tokenization service Docker container on Cloud Run with VPC connector and Workload Identity, so that the service is production-capable quickly without requiring GKE cluster management overhead.
- AC: Cloud Run service deployed from the same Docker image used in all environments; no environment-specific code paths.
- AC: VPC connector configured so all outbound traffic to Spanner, Redis (Memorystore), Cloud KMS, and Pub/Sub routes over private VPC — no public internet egress for data-plane calls.
- AC: Minimum instances set to ≥ 10 for production to eliminate cold-start latency on the hot path.
- AC: Workload Identity configured via Cloud Run service account annotation; no key files present in the container image or build artifacts (SAST validated).
- AC: All existing acceptance criteria from E-01 through E-16 pass on the Cloud Run deployment without modification.

### S-17-02 Customized Approach Documentation For Cloud Run CDE Network Controls
As a compliance engineer, I want a PCI DSS Customized Approach package prepared and QSA-accepted for the Cloud Run compute layer, so that the service can operate in the CDE during Phase 1 without an unmitigated network segmentation gap.
- AC: Customized Approach document covers PCI DSS Requirements 1.2 and 1.3; identifies the security objective (restrict CDE traffic to authorized connections) and describes how the API Gateway ingress controls, VPC connector egress restrictions, and Cloud Armor policies collectively satisfy the objective.
- AC: Document explicitly states that pod/container-level VPC firewall is not available in Cloud Run and describes gateway-layer compensating controls.
- AC: Customized Approach package reviewed and accepted by QSA before first institution is enabled in production on Cloud Run.
- AC: Document stored in compliance evidence repository; referenced in quarterly executive PCI review (E-14).
- AC: Customized Approach is classified as such in the PCI DSS framework — not as a Compensating Control (these are distinct PCI DSS 4.0 constructs).

### S-17-03 GKE Phase 2 Migration — Infrastructure
As a platform engineer, I want to migrate the tokenization service from Cloud Run to a GKE private cluster using the same Docker image, so that VPC-native pod networking natively satisfies PCI DSS 1.2/1.3 and the Customized Approach is retired.
- AC: GKE private cluster provisioned with VPC-native pod networking (alias IP ranges); no public node IPs.
- AC: Kubernetes Network Policy configured to allow only: inbound from API Gateway ingress controller; outbound to Spanner, Redis, KMS, and Pub/Sub private endpoints. All other pod-level egress denied by default.
- AC: Horizontal Pod Autoscaler (HPA) configured targeting the same throughput and latency SLOs as the Cloud Run deployment (50k RPS, p95 ≤ 30ms tokenize, p95 ≤ 35ms detokenize).
- AC: Workload Identity binding updated to GKE `serviceAccountAnnotation` on the Kubernetes ServiceAccount; same GCP service account as Phase 1 — no new IAM grants required.
- AC: Same Docker image used in Phase 1 deployed to GKE without modification; no application code or configuration file changes.
- AC: All acceptance criteria from E-01 through E-16 pass on the GKE deployment without modification.
- AC: Post-migration, the Cloud Run service is disabled (not deleted immediately — retained for one sprint as rollback option, then decommissioned).

### S-17-04 Customized Approach Retirement And Migration Gate
As a compliance engineer, I want the Cloud Run Customized Approach formally retired once GKE is live, and a documented migration readiness gate that evaluates trigger criteria before Phase 2 begins, so that the transition is a planned compliance event rather than an ad hoc infrastructure change.
- AC: Migration readiness checklist documented and approved before Phase 2 IaC work begins. Checklist covers: QSA network segmentation requirement status, institution count vs. 50-institution threshold, platform team GKE readiness self-assessment, Customized Approach maintenance overhead review.
- AC: At least one trigger criterion from §10 Migration Path is formally met and recorded before Phase 2 sprint is scheduled.
- AC: Post-GKE cutover, the Customized Approach package is marked `status=retired` in the compliance evidence repository with the GKE go-live date as the retirement date.
- AC: QSA notified of compute layer change; updated architecture diagram provided showing GKE replacing Cloud Run.
- AC: Quarterly executive PCI review agenda updated to remove Cloud Run network segmentation as an open item.
