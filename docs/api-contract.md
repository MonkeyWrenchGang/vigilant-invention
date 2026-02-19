# API Contract

Base path: `/v1`

## Authentication

- Service-to-service identity via short-lived Google Cloud Workload Identity token. The resolved caller identity is used server-side; callers do not supply a `tenant_id` in the request body — it is derived from the authenticated identity.
- Caller must be authorized for the target `domain` and any provided `scope_qualifiers`.
- De-tokenization additionally requires explicit `detokenize` permission scoped to the caller's service identity + domain + purpose.
- Full PAN return additionally requires the `full-pan` IAM scope granted to the caller identity. Absent this scope, detokenize always returns a masked PAN.

## Observability

- Optional `x-request-id` accepted and echoed in all responses.
- If not provided, service generates one.
- All responses include `request_id` for distributed tracing.

---

## `POST /v1/tokenize`

Create a domain-and-scope-scoped token for a PAN.

### Request

```json
{
  "domain": "checkout",
  "scope_qualifiers": {
    "merchant_id": "m_99821",
    "application_id": "app-mobile-v2"
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
| `domain` | Yes | Business context namespace (e.g., checkout, subscription). |
| `scope_qualifiers` | No | Key-value map of additional scope dimensions (e.g., merchant_id, application_id, channel). All keys and values must be non-PAN string identifiers. |
| `token_purpose` | Yes | Allowed business function enum (e.g., `payment`, `refund`). |
| `token_mode` | Yes | `REUSABLE` or `ONE_TIME`. |
| `pan` | Yes | The Primary Account Number. Never logged or stored plaintext. |
| `exp_month` | No | Card expiry month. |
| `exp_year` | No | Card expiry year. |
| `ttl_seconds` | No | Token TTL override. Defaults to domain policy. |
| `idempotency_key` | Yes | Minimum 8 characters. Scoped to caller identity. |

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

- Idempotency scope: `(caller_identity, idempotency_key)`.
- Repeated requests with same idempotency key return original successful result.
- `REUSABLE` mode may also return a previously generated active token by `(caller_identity, domain, scope_qualifiers_canonical, pan_fingerprint, token_purpose)`.
- `scope_qualifiers` are canonicalized as sorted key-value pairs for lookup consistency.

---

## `POST /v1/detokenize`

Resolve a token to PAN for authorized callers only. Returns masked PAN by default.

### Request

```json
{
  "domain": "checkout",
  "scope_qualifiers": {
    "merchant_id": "m_99821",
    "application_id": "app-mobile-v2"
  },
  "token_purpose": "payment",
  "token": "5R2eW...s8p",
  "request_context": {
    "reason_code": "REFUND_PROCESSING",
    "transaction_id": "tx_550e8400-e29b",
    "operator_id": "user_admin_04"
  },
  "format_options": {
    "return_type": "MASKED_PAN"
  }
}
```

**Fields:**

| Field | Required | Description |
| --- | --- | --- |
| `domain` | Yes | Must match the domain under which the token was issued. |
| `scope_qualifiers` | No | Must match the scope_qualifiers under which the token was issued. |
| `token_purpose` | Yes | Must match the purpose under which the token was issued. |
| `token` | Yes | The token to resolve. |
| `request_context.reason_code` | Yes | Enumerated reason for detokenization. Required for audit trail (PCI DSS 10.2.1.2). Valid values: `PAYMENT_PROCESSING`, `REFUND_PROCESSING`, `FRAUD_INVESTIGATION`, `CHARGEBACK`, `SETTLEMENT`, `COMPLIANCE_REVIEW`. |
| `request_context.transaction_id` | No | Caller-supplied correlation ID linking this call to an originating transaction. Included in audit event. |
| `request_context.operator_id` | No | Caller-asserted identity of the human operator or service context initiating the request. Treated as audit metadata only — not verified by the tokenization service. |
| `format_options.return_type` | No | `MASKED_PAN` (default) or `FULL_PAN`. `FULL_PAN` requires the caller to hold the `full-pan` IAM scope; if the scope is absent the request is rejected with `403 FORBIDDEN`. |

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

### Response 200 — Full PAN (requires `full-pan` scope)

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
- If `scope_qualifiers` in the request do not match the token's issuance scope, the request returns `404 TOKEN_NOT_FOUND` (not a scope mismatch error, to avoid information disclosure).
- `operator_id` is caller-asserted and written to the audit event with a marker indicating it is unverified.

---

## `POST /v1/tokens/revoke`

Revoke active tokens by token value or by scope selector.

### Request — by token

```json
{
  "domain": "checkout",
  "token": "5R2eW...s8p",
  "reason": "merchant_request"
}
```

### Request — by scope fingerprint

```json
{
  "domain": "checkout",
  "scope_qualifiers": {
    "merchant_id": "m_99821"
  },
  "pan_fingerprint": "abc123...",
  "reason": "fraud_signal"
}
```

`scope_qualifiers` in the fingerprint revocation selector acts as a filter. Omitting it revokes all matching tokens for the fingerprint within the domain regardless of scope. Including it restricts revocation to tokens matching both domain and the provided qualifier values.

**Reason codes:** `merchant_request`, `fraud_signal`, `card_compromised`, `merchant_offboarded`, `application_decommissioned`, `compliance_action`.

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
  "message": "domain not allowed for service identity",
  "request_id": "9bc7..."
}
```

Status mapping:

| HTTP Status | Error Code | When Used |
| --- | --- | --- |
| `400` | `INVALID_REQUEST` | Malformed request, missing required fields, invalid enum value |
| `401` | `UNAUTHORIZED` | Missing or invalid caller identity token |
| `403` | `FORBIDDEN` | Caller identity lacks required permission for domain, scope, purpose, or full-pan |
| `404` | `TOKEN_NOT_FOUND` | Token does not exist, is revoked, or scope mismatch (unified to avoid disclosure) |
| `409` | `CONFLICT` | Idempotency key reuse with different parameters |
| `500` | `INTERNAL_ERROR` | Unhandled service fault |

Error responses never include PAN, CVV, key material, stack traces, or internal system identifiers.
