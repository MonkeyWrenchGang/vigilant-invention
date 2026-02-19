# PCI Security Controls Matrix

| Control Area | Requirement | Implementation | Evidence Artifact |
| --- | --- | --- | --- |
| Key management | Strong cryptographic key lifecycle and separation of duties (PCI DSS 3.6, 3.7) | Cloud KMS key ring with HSM-backed keys; two distinct key classes: PEK (envelope encryption) and PFK (HMAC fingerprint); 90-day rotation; key custodian IAM identity strictly separated from data-plane access | KMS key metadata export, IAM policy snapshots (quarterly), rotation run logs, key/data separation audit report |
| Data encryption at rest | PAN data must be encrypted at field/application layer; disk-level encryption alone is insufficient (PCI DSS 3.4.1, 3.5.1.2) | Field-level AES-256-GCM envelope encryption applied at application layer before writes to Spanner vault tables. Storage-layer encryption is defense-in-depth only. | Encryption design doc, integration test evidence, schema review, SAST confirmation of no plaintext PAN writes |
| PAN fingerprint integrity | Keyed hash required for indexed PAN lookups; unkeyed SHA-256 prohibited (PCI DSS 3.5.1.1) | HMAC-SHA-256 computed using Cloud KMS-managed PAN Fingerprint Key (PFK); PFK is institution-scoped (one per institution), HSM-backed, and distinct from PEK | HMAC implementation test evidence, KMS PFK key metadata, code review confirming no unkeyed hash path |
| Data encryption in transit | Encrypt all traffic carrying CHD | TLS 1.2+ minimum (TLS 1.3 preferred) for ingress; internal service traffic over private networking | TLS config output, API gateway settings, penetration test report |
| Access control | Restrict access by role/service identity; de-tokenize requires explicit scope permission (PCI DSS 7) | Workload Identity + service identity header checks for domain/scope_qualifiers/purpose and detokenize permission; hard-coded credentials prohibited; Workload Identity enforced for runtime | IAM role matrix, authz policy tests, access review records, SAST scan confirming no hardcoded credentials |
| Authentication — service | Strong service authN (PCI DSS 8.6) | Google Cloud Workload Identity for all JH caller applications (Banno, SilverLake, Symitar, ProfitStars); no long-lived static service account key files in production; delegation grant table enforces per-institution authorization | Workload Identity config export, delegation grant table audit, SAST confirmation of no static credential files |
| Authentication — human (MFA) | MFA required for all human CDE access (PCI DSS 4.0 Req 8.4.2, 8.3.6) | Google Workspace MFA enforcement policy applied to all identities with CDE-scoped resource access; minimum 12-character passwords; MFA cannot be bypassed including break-glass | IdP MFA enforcement policy export, access review confirmation, break-glass procedure audit |
| Audit logging | Record all access to CHD/token operations including reason code (PCI DSS 10.2.1.2) | Immutable audit event stream to Pub/Sub and warehouse sink; includes request id, actor, action, outcome, timestamp, reason_code (detokenize), and scope qualifiers | Audit samples, retention config, SIEM dashboard export |
| Log redaction | Prevent PAN/CVV in logs | Structured logging middleware with denylist redaction and payload validation | Log policy config, redaction unit tests, sample sanitized logs |
| Automated PAN data discovery | Detective scanning for out-of-vault PAN presence (PCI DSS 4.0 Req 12.5.2) | Automated discovery tool scanning Cloud Logging buckets, BigQuery audit tables, error reporting, Spanner exports, and CI/CD artifacts at minimum annually and after significant infrastructure changes | Discovery scan reports, remediation records, scan schedule attestation |
| Data retention | Defined retention and secure disposal | Time-bound retention for audit and token state, automated lifecycle rules | Retention policy docs, storage lifecycle settings |
| Secure SDLC | Vulnerability and code quality controls; OWASP Top 10 compliance (PCI DSS 6.2.4) | CI pipeline with SAST/dependency scans, policy checks, test gates; OWASP Top 10 review covering injection, insecure deserialization, broken access control, and security misconfiguration at each release | Pipeline configuration, OWASP test evidence bundle, scan reports, release approvals |
| Authenticated vulnerability scanning | Internal authenticated scans required (PCI DSS 11.3.1.2) | Quarterly authenticated internal scans (credentialed OS + application layer); unauthenticated external and internal scans as separate tiers; evidence distinguishes all three tiers | Authenticated scan reports (quarterly), external scan reports, remediation records |
| Incident response | Detect and respond to security events | Alerting for authz failures, unusual detokenize spikes, key access anomalies, and PAN discovery findings | Incident runbook, drill records, postmortem templates |
| Change management | Controlled production change process | Canary deployment with rollback thresholds and approval workflow | Release checklist, deployment logs, rollback exercise evidence |
| Business continuity | Recover within RTO/RPO objectives | Backup/export strategy for vault metadata and replayable audit streams | Recovery test report, backup schedules, restore validation |
| Executive accountability | Executive management formally owns PCI compliance (PCI DSS 4.0 Req 12.4.1) | Quarterly executive compliance review; documented charter with named executive owner | Quarterly review records, compliance charter, sign-off artifacts |
| Customer responsibility acknowledgment | Written service provider responsibility statement required (PCI DSS 4.0 Req 12.9.1) | Shared Responsibility Agreement provided to each institution at onboarding; reviewed annually | Executed agreements per institution, agreement template, annual re-execution records |

## Evidence Collection Checklist

- Monthly IAM access review records.
- Quarterly key rotation drill and attestation (PEK and PFK separately).
- Quarterly key/data separation of duties IAM snapshot validation.
- Quarterly authenticated internal vulnerability scan and remediation records.
- Quarterly executive PCI compliance review minutes.
- Continuous audit pipeline health dashboard.
- Redaction compliance checks against sampled logs.
- Annual Shared Responsibility Agreement review and re-execution per institution.
- Annual automated PAN data discovery scan (plus ad-hoc after infrastructure changes).
- OWASP Top 10 test evidence bundle for each release candidate.
- SDLC evidence bundle for each release candidate.
- Incident simulations for detokenize abuse and key compromise.
