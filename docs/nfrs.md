# Non-Functional Requirements (NFRs)

## Service Scope

- System: PCI tokenization and de-tokenization platform.
- Stack: GCP + Python.
- Topology: single region, multi-zone high availability.
- Data class: PCI-sensitive PAN data in vault boundary.

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
- Per-tenant fairness:
  - Base quota: 500 RPS per tenant.
  - Burst quota: 2,000 RPS per tenant for 60 seconds.
- Hard ceiling:
  - 10,000 RPS per tenant unless explicitly approved.
  - 5,000 RPS per domain unless explicitly approved.

## Security And Compliance NFRs

- PAN data encrypted at rest using envelope encryption (AEAD).
- Keys managed in Cloud KMS with HSM-backed protection for key-encryption keys.
- No PAN/CVV in logs, metrics labels, traces, or error bodies.
- Strict authZ for de-tokenization by service identity + domain + purpose.
- Full audit record for tokenization, de-tokenization, revocation, and failed attempts.

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

- Load tests show sustained 50k RPS while meeting p95/p99 SLOs.
- Chaos/failure tests prove fail-closed behavior for de-tokenize on authZ degradation.
- Synthetic probes validate health/readiness every 30 seconds from at least 3 zones.
