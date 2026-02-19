# Non-Functional Requirements (NFRs)

## Service Scope

- System: Jack Henry central PCI tokenization and de-tokenization platform.
- Stack: GCP + Python. Compute: GKE private cluster (VPC-native pod networking satisfies PCI DSS 1.2/1.3 without Customized Approach).
- Topology: single region, multi-zone high availability (multi-region active-active is a Future Phase item).
- Data class: PCI-sensitive PAN data in vault boundary, partitioned by `institution_id`.
- Callers: Jack Henry internal application services (Banno, SilverLake, Symitar, ProfitStars) acting on behalf of bank and credit union institution clients.

## Throughput And Latency Targets

### API Throughput

- Peak sustained throughput: 50,000 requests per second aggregate.
- Burst throughput: 60,000 requests per second for 5 minutes.
- Default traffic split for testing:
  - 70% `POST /v1/tokenize`
  - 25% `POST /v1/detokenize`
  - 5% `POST /v1/tokens/revoke`

### Latency SLOs (service-side)

- `POST /v1/tokenize`
  - p95 <= 30 ms
  - p99 <= 60 ms
- `POST /v1/detokenize`
  - p95 <= 35 ms
  - p99 <= 70 ms
- `POST /v1/tokens/revoke`
  - p95 <= 50 ms
  - p99 <= 120 ms

Notes:
- These targets exclude client internet latency.
- SLOs assume healthy cache hit rates and no control-plane incidents.

## Availability And Durability

- Monthly availability target:
  - Tokenize API: 99.95%
  - De-tokenize API: 99.95%
- Data durability target for token vault metadata: 99.999999999%.
- RTO <= 30 minutes.
- RPO <= 5 minutes (regional failure events handled by restore/replay).

## Error Budget

- 99.95% monthly SLO allows ~21.9 minutes monthly downtime.
- Burn-rate alerts:
  - Fast burn: 14x over 1 hour page to on-call.
  - Slow burn: 2x over 6 hours ticket + investigation.

## Concurrency Model

- Concurrent active client connections target: 100,000.
- Per-institution rate limits (protects service fairness across bank/credit union clients):
  - Base quota: 500 RPS per institution.
  - Burst quota: 2,000 RPS per institution for 60 seconds.
  - Hard ceiling: 10,000 RPS per institution unless explicitly approved by platform team.
- Per-caller-application rate limits (protects against single JH product line saturation):
  - Default: 5,000 RPS per caller application (e.g., Banno aggregate, SilverLake aggregate).
  - Hard ceiling: requires platform team approval.
- Rate limit buckets are institution-isolated: one institution's burst cannot consume another institution's quota.

## Security And Compliance NFRs

- PAN data encrypted at rest using AES-256-GCM envelope encryption (field-level, not disk-level).
- Two KMS key classes per institution: PFK (HMAC-SHA-256 for fingerprinting) and PEK (AES-256-GCM for encryption). Both are institution-scoped (one per institution, provisioned at onboarding).
- Keys managed in Cloud KMS with HSM-backed protection; 90-day rotation.
- No PAN/CVV in logs, metrics labels, traces, or error bodies.
- All callers authenticated via Workload Identity; no static API keys.
- Authorization: Workload Identity validity → delegation grant for `institution_id` → domain/purpose policy → (detokenize) full-pan scope gate.
- Full audit record for tokenization, de-tokenization, revocation, and failed attempts, with `caller_application` and `institution_id` as mandatory indexed fields.
- Institution Registry lookup: ≤ 2ms added to request latency (in-process cached).
- Delegation grant check: ≤ 1ms added to request latency (in-process cached).

## Operational NFRs

- Cold-start mitigation enabled on API compute layer.
- Autoscaling reaction time <= 30 seconds for sustained load changes.
- Deployments use canary strategy with automatic rollback triggers.
- End-to-end traceability via request ID across gateway, API, store, and audit sinks.

## Capacity Guardrails

- Redis target hit ratio >= 85% for hot reads.
- Spanner CPU utilization target <= 65% steady-state.
- API CPU utilization target <= 70% steady-state.
- Queue backlog and dead-letter queues monitored with alerting thresholds.

## Validation Acceptance Criteria

- Load tests show sustained 50k RPS while meeting p95/p99 SLOs under multi-institution, multi-caller-application traffic mix.
- Rate limiting validated: per-institution and per-caller-application quotas enforced independently.
- Chaos/failure tests prove fail-closed behavior for de-tokenize on delegation grant service degradation — no PAN released.
- Synthetic probes validate health/readiness every 30 seconds from at least 3 zones.
- Institution Registry and delegation grant check latency contributions measured and within NFR budgets under load.
