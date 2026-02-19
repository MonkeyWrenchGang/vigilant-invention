# Performance Validation And Tuning

## Test Stages

- Stage 1: 10k RPS steady for 2 minutes.
- Stage 2: 25k RPS steady for 2 minutes.
- Stage 3: 50k RPS steady for 2 minutes with mixed tokenize/detokenize workload.

Tools:
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
- Warm service before load test.
- Execute load generation from separate VPC project to avoid resource contention.
- Record:
  - p50/p95/p99 latencies by endpoint
  - error rate by status code
  - Cloud Run CPU/memory/concurrency
  - Spanner CPU/read/write latency
  - Redis hit/miss ratio

## Tuning Checklist

- API:
  - Increase Cloud Run max instances and concurrency as needed.
  - Tune Python worker count and request timeout.
- Store:
  - Confirm Spanner indexes support reusable lookup and revocation scans.
  - Keep Spanner CPU <= 65% at steady peak.
- Cache:
  - Maintain Redis hit ratio >= 85% for hot reads.
  - Set TTL strategy by token mode and revocation policy.

## Acceptance Criteria

- Meet or beat NFR SLOs at 50k RPS class:
  - Tokenize p95 <= 30 ms, p99 <= 60 ms
  - Detokenize p95 <= 35 ms, p99 <= 70 ms
- Error rate < 1%.
- No unauthorized de-tokenize success paths.

## Current Execution Note

- Load scripts and scenarios are implemented and ready to run.
- Local staged smoke runs completed with Locust headless mode:
  - Stage A (`-u 100`, 20s): ~1340 req/s aggregate, 0% failures.
  - Stage B (`-u 250`, 20s): ~1310 req/s aggregate, 0% failures.
  - Stage C (`-u 500`, 20s): ~1284 req/s aggregate, 0% failures.
- Full 50k RPS validation must run in GCP staging with distributed load generators and production-like networking.
