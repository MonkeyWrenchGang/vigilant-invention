# API Contract

Base path: `/v1`

## Context: Jack Henry Internal Central Tokenization Service

This API is consumed exclusively by Jack Henry internal application services (Banno, SilverLake, Symitar, ProfitStars, and others). It is not exposed to institution clients or cardholders directly. All callers are Jack Henry-operated services running within the Jack Henry GCP organization.

---

## Authentication And Authorization

### Caller Identity
Caller application identity is established via **Google Cloud Workload Identity**. Each Jack Henry application service has a dedicated service account (e.g., `banno-payments@jh-payments.iam.gserviceaccount.com`). No static API keys or long-lived credentials are used.

### Institution Authorization (Delegation Model)
Every request supplies an `institution_id` identifying the bank or credit union client on whose behalf the operation is being performed. The authorization policy validates that the caller application's service account is **delegated** to act for that institution. A caller supplying an unauthorized `institution_id` receives `403 FORBIDDEN` regardless of the operation.

```
Authorization check order:
1. Is the caller's Workload Identity valid?          → 401 if not
2. Is the caller authorized for this institution_id? → 403 if not
3. Is the caller authorized for this domain/purpose? → 403 if not
4. (detokenize only) Does caller hold full-pan scope? → 403 if not and FULL_PAN requested
```

### Observability
- Optional `x-request-id` header accepted and echoed in all responses.
- If not provided, service generates one.
- All responses include `request_id` for distributed tracing.

---

## `POST /v1/tokenize`

Create a token for a PAN, scoped to an institution, domain, and optional scope qualifiers.

### Request

```json
{
  "institution_id": "inst_firstnational_001",
  "domain": "card-payments",
  "scope_qualifiers": {
    "application": "banno-mobile",
    "channel": "web"
  },
  "token_purpose": "payment",
  "token_mode": "REUSABLE",
  "pan": "4111111111111111",
  "exp_month": 12,
  "exp_year": 2030,
  "ttl_seconds": 3600,
  "idempotency_key": "8+ char unique key"
}
```

**Fields:**

| Field | Required | Description |
| --- | --- | --- |
| `institution_id` | Yes | Jack Henry institution identifier. Caller must be authorized to act for this institution. |
| `domain` | Yes | Business context namespace within the institution (e.g., `card-payments`, `ach`, `digital-banking`). |
| `scope_qualifiers` | No | Key-value map for further differentiation within domain (e.g., `{"application": "banno-mobile", "channel": "web"}`). Keys and values must be non-PAN strings. |
| `token_purpose` | Yes | Enumerated business function: `payment`, `refund`, `verification`, `provisioning`. |
| `token_mode` | Yes | `REUSABLE` or `ONE_TIME`. |
| `pan` | Yes | Primary Account Number. Never logged or stored plaintext. |
| `exp_month` | No | Card expiry month. |
| `exp_year` | No | Card expiry year. |
| `ttl_seconds` | No | Token TTL override. Defaults to domain policy. |
| `idempotency_key` | Yes | Minimum 8 characters. Scoped to `(caller_identity, institution_id)`. |

### Response 200

```json
{
  "token": "5R2eW...s8p",
  "token_mode": "REUSABLE",
  "token_state": "ACTIVE",
  "expires_at": "2026-02-17T12:30:00Z",
  "reused_existing": true,
  "request_id": "9bc7..."
}
```

### Idempotency Behavior

- Idempotency scope: `(caller_identity, institution_id, idempotency_key)`.
- Repeated requests with the same key combination return the original successful result.
- `REUSABLE` mode additionally deduplicates by `(institution_id, domain, scope_qualifiers_canonical, token_purpose, pan_fingerprint)` — if an active token exists for that scope, it is returned with `reused_existing: true`.
- `scope_qualifiers` are canonicalized as lexicographically sorted key-value pairs before lookup.

---

## `POST /v1/detokenize`

Resolve a token to PAN for authorized callers. Returns masked PAN by default.

### Request

```json
{
  "institution_id": "inst_firstnational_001",
  "domain": "card-payments",
  "scope_qualifiers": {
    "application": "banno-mobile",
    "channel": "web"
  },
  "token_purpose": "payment",
  "token": "5R2eW...s8p",
  "request_context": {
    "reason_code": "REFUND_PROCESSING",
    "transaction_id": "tx_550e8400-e29b",
    "operator_id": "bank-employee-id-04"
  },
  "format_options": {
    "return_type": "MASKED_PAN"
  }
}
```

**Fields:**

| Field | Required | Description |
| --- | --- | --- |
| `institution_id` | Yes | Must match the institution under which the token was issued. |
| `domain` | Yes | Must match the domain under which the token was issued. |
| `scope_qualifiers` | No | Must match the scope_qualifiers under which the token was issued. |
| `token_purpose` | Yes | Must match the purpose under which the token was issued. |
| `token` | Yes | The token to resolve. |
| `request_context.reason_code` | Yes | Required enumerated reason. Valid values: `PAYMENT_PROCESSING`, `REFUND_PROCESSING`, `FRAUD_INVESTIGATION`, `CHARGEBACK`, `SETTLEMENT`, `COMPLIANCE_REVIEW`. Requests without a valid reason code are rejected with `400`. |
| `request_context.transaction_id` | No | Caller-supplied correlation ID to the originating transaction. Written to audit event. |
| `request_context.operator_id` | **Required for `FULL_PAN`**; optional for `MASKED_PAN` | Caller-asserted identifier of the bank employee or institution end-user initiating the request. Required when requesting full cleartext PAN to create an attribution chain for the highest-risk operation. Treated as unverified audit metadata — written to the audit event with an `unverified` flag. |
| `format_options.return_type` | No | `MASKED_PAN` (default) or `FULL_PAN`. `FULL_PAN` requires the caller application to hold the `full-pan` IAM scope; absent the scope, request is rejected with `403 FORBIDDEN`. `FULL_PAN` also requires `operator_id`. |

### Response 200 — Masked (default)

```json
{
  "token": "5R2eW...s8p",
  "pan": "411111******1111",
  "exp_month": 12,
  "exp_year": 2030,
  "request_id": "9bc7..."
}
```

### Response 200 — Full PAN (requires `full-pan` IAM scope)

```json
{
  "token": "5R2eW...s8p",
  "pan": "4111111111111111",
  "exp_month": 12,
  "exp_year": 2030,
  "request_id": "9bc7..."
}
```

### Detokenize Behavior Notes

- Service fails closed on any authorization uncertainty or policy service error — no fallback to allow.
- `institution_id` or `scope_qualifiers` mismatch returns `404 TOKEN_NOT_FOUND` (same as token not found) to avoid disclosing whether a token exists under a different institution or scope.
- `operator_id` is caller-asserted and written to the audit event with `"verified": false`. It is not authenticated by the tokenization service. It is required for `FULL_PAN` requests to ensure every cleartext PAN access has an attributed bank employee or operator identity in the audit record.

---

## `POST /v1/tokens/revoke`

Revoke active tokens by token value or by institution fingerprint selector.

### Request — by token

```json
{
  "institution_id": "inst_firstnational_001",
  "domain": "card-payments",
  "token": "5R2eW...s8p",
  "reason": "institution_request"
}
```

### Request — by fingerprint selector

```json
{
  "institution_id": "inst_firstnational_001",
  "domain": "card-payments",
  "scope_qualifiers": {
    "application": "banno-mobile"
  },
  "pan_fingerprint": "abc123...",
  "reason": "card_reissued"
}
```

`scope_qualifiers` in the fingerprint selector acts as a filter:
- **Provided**: revokes only tokens matching both the domain and all supplied qualifier key-value pairs (e.g., revoke all Banno-issued tokens for this card at this institution).
- **Omitted**: revokes all active tokens for the fingerprint within the institution and domain.

**Reason codes:** `institution_request`, `fraud_signal`, `card_compromised`, `card_reissued`, `application_decommissioned`, `compliance_action`.

### Response 200

```json
{
  "revoked_count": 2,
  "request_id": "9bc7..."
}
```

---

## Errors

Uniform error body:

```json
{
  "error_code": "FORBIDDEN",
  "message": "caller not authorized for institution",
  "request_id": "9bc7..."
}
```

| HTTP Status | Error Code | When Used |
| --- | --- | --- |
| `400` | `INVALID_REQUEST` | Malformed request, missing required fields, invalid enum value, PAN-like value in scope_qualifiers |
| `401` | `UNAUTHORIZED` | Missing or invalid Workload Identity token |
| `403` | `FORBIDDEN` | Caller not authorized for institution_id, domain, purpose, or full-pan scope |
| `404` | `TOKEN_NOT_FOUND` | Token does not exist, is revoked, or institution/scope mismatch (unified — no disclosure) |
| `409` | `CONFLICT` | Idempotency key reuse with different parameters |
| `429` | `RATE_LIMITED` | Per-institution or per-caller-application quota exceeded |
| `500` | `INTERNAL_ERROR` | Unhandled service fault |

Error responses never include PAN, CVV, key material, stack traces, or internal system identifiers.

---

## Institution Onboarding Requirements

Before an institution's data can be processed:
1. Institution-scoped PFK must be provisioned in Cloud KMS (HMAC-SHA-256, HSM-backed).
2. Institution-scoped PEK must be provisioned in Cloud KMS (AES-256-GCM, HSM-backed). Both keys are required; no shared-PEK option.
3. Institution `institution_id` must be registered in the Institution Registry (Spanner config table) with PFK and PEK key references.
4. Caller application(s) must be granted delegation authorization for the institution in the grant table.

Onboarding is an operational procedure; it is not automated via this API.
