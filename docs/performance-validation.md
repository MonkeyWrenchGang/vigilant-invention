# Performance Validation And Tuning

## Context

The tokenization service is a Jack Henry internal platform consumed by Banno, SilverLake, Symitar, and ProfitStars on behalf of bank and credit union institution clients. Load test scenarios must reflect:

- Multiple JH caller applications generating concurrent traffic.
- Multiple institutions (`institution_id`) served in parallel — institution isolation must hold under load.
- Institution Registry and delegation grant lookups as latency contributors.
- Redis cache keyed by `institution_id` as the leading dimension.
- Per-institution and per-caller-application rate limit buckets exercised independently.

## Test Stages

- Stage 1: 10k RPS steady for 2 minutes.
- Stage 2: 25k RPS steady for 2 minutes.
- Stage 3: 50k RPS steady for 2 minutes with mixed tokenize/detokenize workload.

Default traffic split at each stage:

- 70% `POST /v1/tokenize`
- 25% `POST /v1/detokenize`
- 5% `POST /v1/tokens/revoke`

## Caller And Institution Traffic Mix

Load tests must simulate a realistic Jack Henry traffic profile:

| Simulated Caller   | Share | Institutions  | Notes                                     |
| ------------------ | ----- | ------------- | ----------------------------------------- |
| Banno              | 50%   | 10–50         | Mix of REUSABLE and ONE_TIME modes        |
| SilverLake         | 30%   | 5–20          | Predominantly REUSABLE                    |
| Symitar            | 15%   | 3–10          | Mix of domains (card-payments, ach)       |
| ProfitStars        | 5%    | 1–5           | Low-volume, detokenize-heavy              |

- Each caller uses a distinct simulated Workload Identity service account.
- Each institution has a unique `institution_id` in the test registry with pre-provisioned test PFK and PEK.
- `scope_qualifiers` varies across requests: `{"application": "banno-mobile"}`, `{"application": "silverlake-core", "channel": "web"}`, absent.

## Tools

- k6 scenario runner: `load/k6/tokenization.js`
- Locust user simulation: `load/locustfile.py`

## Local Execution

1. Start API:
   - `uvicorn app.main:app --host 0.0.0.0 --port 8000`
2. Run k6:
   - `BASE_URL=http://127.0.0.1:8000 k6 run load/k6/tokenization.js`
3. Run Locust:
   - `locust -f load/locustfile.py --host http://127.0.0.1:8000`

## Production-Like Execution (GCP)

- Deploy service with min instances >= 10 and CPU always allocated.
- Warm service before load test (Institution Registry and delegation grant caches must be populated).
- Seed test institutions in Institution Registry and delegation grant table before test run.
- Execute load generation from separate VPC project to avoid resource contention.
- Record:
  - p50/p95/p99 latencies by endpoint and by caller application
  - Error rate by status code — watch for `429 RATE_LIMITED` indicating quota misconfiguration
  - Delegation grant denial rate (`403 FORBIDDEN`) — should be 0% for authorized callers
  - GKE pod/node CPU, memory, and concurrency per replica
  - Spanner CPU/read/write latency per institution partition
  - Redis hit/miss ratio overall and per institution
  - Institution Registry lookup latency (target ≤ 2ms contribution)
  - Delegation grant check latency (target ≤ 1ms contribution)

## Tuning Checklist

- API:
  - Adjust GKE HPA min/max replica counts and target CPU utilization as needed.
  - Tune Python worker count and request timeout per pod.
  - Verify in-process Institution Registry cache TTL (60s recommended) is populated before load ramps.
- Store:
  - Confirm Spanner indexes lead with `institution_id` for all lookup and revocation scans.
  - Keep Spanner CPU ≤ 65% at steady peak; no cross-institution index fan-out.
- Cache:
  - Maintain Redis hit ratio ≥ 85% for hot reads.
  - Confirm Redis cache key format: `{institution_id}:{domain}:{scope_qualifiers_canonical}:{purpose}:{pan_fingerprint}`.
  - Set TTL strategy by token mode and revocation policy.
- Rate limits:
  - Verify per-institution rate limit bucket isolation: one institution's burst does not consume another's quota.
  - Verify per-caller-application limits are enforced independently of institution limits.

## Rate Limit Validation Scenario

Run a targeted rate limit test alongside the main load test:

1. Ramp institution A to 2,500 RPS (beyond its 2,000 RPS burst ceiling).
2. Confirm institution A receives `429 RATE_LIMITED` beyond ceiling.
3. Confirm institution B at 400 RPS is unaffected — 0% `429` responses.
4. Confirm Banno application at 5,500 RPS aggregate receives `429` beyond its 5,000 RPS default.

## Acceptance Criteria

- Meet or beat NFR SLOs at 50k RPS class:
  - Tokenize p95 ≤ 30 ms, p99 ≤ 60 ms
  - Detokenize p95 ≤ 35 ms, p99 ≤ 70 ms
- Error rate < 1% (excluding expected `429` from rate limit test).
- No cross-institution data served: chaos test verifies institution A's tokens never returned for institution B requests.
- No unauthorized detokenize success paths — delegation grant failure returns `403`, never PAN.
- Institution Registry lookup ≤ 2ms and delegation grant check ≤ 1ms confirmed under load.

## Current Execution Note

- Load scripts and scenarios are implemented and ready to run.
- Local staged smoke runs completed with Locust headless mode:
  - Stage A (`-u 100`, 20s): ~1340 req/s aggregate, 0% failures.
  - Stage B (`-u 250`, 20s): ~1310 req/s aggregate, 0% failures.
  - Stage C (`-u 500`, 20s): ~1284 req/s aggregate, 0% failures.
- Local runs use a single test institution; multi-institution and multi-caller-app scenarios require GCP staging with distributed load generators, pre-seeded Institution Registry, and production-like networking.
