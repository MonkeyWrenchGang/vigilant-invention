# Architecture And Event Sequence Diagrams

## System Fit Diagram

```mermaid
flowchart LR
  subgraph clients [ClientsAndConsumers]
    clientApps[ClientApps]
  end

  subgraph edge [IngressAndAPI]
    apiGateway[API_Gateway]
    tokenApi[TokenizationAPI_CloudRun]
  end

  subgraph identity [IdentityAndPolicy]
    workloadIdentity[WorkloadIdentity_GCP]
    authPolicy[AuthZ_Policy\ndomain+scope+purpose]
  end

  subgraph crypto [CryptographicLayer]
    pemKey[CloudKMS_PEK\nAES256_Envelope]
    pfkKey[CloudKMS_PFK\nHMAC_SHA256_Fingerprint]
  end

  subgraph data [VaultAndDataPlane]
    redisCache[Redis_HotCache]
    spannerVault[Spanner_TokenVault\nscope_qualifiers_indexed]
    secretManager[SecretManager]
  end

  subgraph observability [AuditAndObservability]
    pubsubAudit[PubSub_AuditEvents]
    bigqueryAudit[BigQuery_AuditStore]
    monitoring[CloudMonitoring_Alerts]
    panDiscovery[PAN_DiscoveryScanner\nSR-08_Annual]
  end

  clientApps -->|WorkloadIdentity token| apiGateway
  apiGateway --> tokenApi
  tokenApi --> workloadIdentity
  tokenApi --> authPolicy
  tokenApi --> pfkKey
  tokenApi --> pemKey
  tokenApi --> redisCache
  tokenApi --> spannerVault
  tokenApi --> secretManager
  tokenApi --> pubsubAudit
  pubsubAudit --> bigqueryAudit
  tokenApi --> monitoring
  panDiscovery -.->|scans| bigqueryAudit
  panDiscovery -.->|scans| spannerVault
```

**Key architectural properties:**
- Caller tenant is resolved from Workload Identity — not from request body.
- Two KMS key classes: PEK for AES-256-GCM PAN encryption; PFK for HMAC-SHA-256 fingerprint only. Never interchangeable.
- PAN discovery scanner (SR-08) runs out-of-band, scanning audit and storage targets for out-of-vault PAN presence.
- All human access to any component in the identity, crypto, and data subgraphs requires MFA (SR-07).

---

## Tokenize Sequence

```mermaid
sequenceDiagram
  participant Client as ClientService
  participant Gateway as API_Gateway
  participant API as TokenizationAPI
  participant WI as WorkloadIdentity
  participant Policy as AuthZ_Policy
  participant Redis as Redis_Cache
  participant PFK as CloudKMS_PFK
  participant PEK as CloudKMS_PEK
  participant Spanner as Spanner_Vault
  participant Audit as PubSub_Audit

  Client->>Gateway: POST /v1/tokenize {domain, scope_qualifiers, token_purpose, token_mode, pan, idempotency_key}
  Gateway->>API: ForwardRequest
  API->>WI: ResolveCallerIdentity_and_Tenant
  WI-->>API: TenantId + CallerIdentity
  API->>Policy: ValidateIdentity_Domain_Scope_Purpose
  Policy-->>API: AllowOrDeny

  alt REUSABLE mode
    API->>PFK: HMAC_SHA256(PAN, tenant_pfk)
    PFK-->>API: pan_fingerprint
    API->>Redis: LookupToken {tenant, domain, scope_qualifiers_canonical, purpose, pan_fingerprint}
    alt cache hit
      Redis-->>API: ActiveToken
      API->>Audit: PublishTokenizeEvent {reused=true, domain, scope_qualifiers, purpose}
      API-->>Gateway: 200 TokenResponse
      Gateway-->>Client: 200 TokenResponse
    else cache miss
      Redis-->>API: Miss
      API->>Spanner: LookupReusableToken {tenant, domain, scope_qualifiers_canonical, purpose, pan_fingerprint, state=ACTIVE}
      alt vault hit
        Spanner-->>API: ActiveTokenRecord
        API->>Redis: WriteThrough
        API->>Audit: PublishTokenizeEvent {reused=true}
        API-->>Gateway: 200 TokenResponse
        Gateway-->>Client: 200 TokenResponse
      else vault miss — mint new token
        API->>PEK: EncryptPAN_AES256GCM(pan)
        PEK-->>API: CiphertextAndMetadata
        API->>Spanner: UpsertCardRecord + CreateTokenRecord {domain, scope_qualifiers_canonical, purpose}
        Spanner-->>API: TokenRecord
        API->>Redis: WriteThrough
        API->>Audit: PublishTokenizeEvent {reused=false}
        API-->>Gateway: 200 TokenResponse
        Gateway-->>Client: 200 TokenResponse
      end
    end
  else ONE_TIME mode
    API->>PFK: HMAC_SHA256(PAN, tenant_pfk)
    PFK-->>API: pan_fingerprint
    API->>PEK: EncryptPAN_AES256GCM(pan)
    PEK-->>API: CiphertextAndMetadata
    API->>Spanner: UpsertCardRecord + CreateTokenRecord
    Spanner-->>API: TokenRecord
    API->>Audit: PublishTokenizeEvent {mode=ONE_TIME}
    API-->>Gateway: 200 TokenResponse
    Gateway-->>Client: 200 TokenResponse
  end
```

---

## Detokenize Sequence

```mermaid
sequenceDiagram
  participant Client as ClientService
  participant Gateway as API_Gateway
  participant API as TokenizationAPI
  participant WI as WorkloadIdentity
  participant Policy as AuthZ_Policy
  participant Redis as Redis_Cache
  participant PEK as CloudKMS_PEK
  participant Spanner as Spanner_Vault
  participant Audit as PubSub_Audit

  Client->>Gateway: POST /v1/detokenize {domain, scope_qualifiers, token_purpose, token, request_context{reason_code,...}, format_options}
  Gateway->>API: ForwardRequest
  API->>WI: ResolveCallerIdentity_and_Tenant
  WI-->>API: TenantId + CallerIdentity
  API->>API: ValidateReasonCode (reject if missing or invalid enum)
  API->>Policy: ValidateDetokenizePermission {identity, domain, scope_qualifiers, purpose}
  Policy-->>API: AllowOrDeny
  API->>Policy: CheckFullPanScope (if return_type=FULL_PAN)
  Policy-->>API: AllowOrDeny

  note over API: Fail closed on any policy error — no fallback

  API->>Redis: LookupTokenToCard {token}
  alt cache hit
    Redis-->>API: CardReference
  else cache miss
    API->>Spanner: ReadTokenAndCard {token, tenant, domain, scope_qualifiers}
    Spanner-->>API: VaultRecord
    API->>Redis: PopulateHotCache
  end

  API->>PEK: DecryptPAN_AES256GCM(ciphertext)
  PEK-->>API: PlaintextPAN

  alt return_type = MASKED_PAN (default)
    API->>API: MaskPAN (411111******1111)
    API->>Audit: PublishDetokenizeEvent {reason_code, operator_id=unverified, return_type=MASKED, domain, scope_qualifiers}
    API-->>Gateway: 200 {token, pan=masked, exp_month, exp_year}
    Gateway-->>Client: 200 MaskedPANResponse
  else return_type = FULL_PAN (full-pan scope confirmed)
    API->>Audit: PublishDetokenizeEvent {reason_code, operator_id=unverified, return_type=FULL, domain, scope_qualifiers}
    API-->>Gateway: 200 {token, pan=full, exp_month, exp_year}
    Gateway-->>Client: 200 FullPANResponse
  end
```

---

## Revoke Sequence

```mermaid
sequenceDiagram
  participant Client as ClientService
  participant Gateway as API_Gateway
  participant API as TokenizationAPI
  participant WI as WorkloadIdentity
  participant Policy as AuthZ_Policy
  participant Spanner as Spanner_Vault
  participant Redis as Redis_Cache
  participant Audit as PubSub_Audit

  Client->>Gateway: POST /v1/tokens/revoke {domain, token OR pan_fingerprint+scope_qualifiers, reason}
  Gateway->>API: ForwardRequest
  API->>WI: ResolveCallerIdentity_and_Tenant
  WI-->>API: TenantId + CallerIdentity
  API->>Policy: ValidateRevokePermission {identity, domain}
  Policy-->>API: AllowOrDeny

  alt revoke by token
    API->>Spanner: SetTokenState {token → REVOKED}
    Spanner-->>API: UpdatedCount
    API->>Redis: EvictToken {token}
  else revoke by fingerprint selector
    API->>Spanner: SetTokensRevoked {tenant, domain, pan_fingerprint, scope_qualifiers_filter, state=ACTIVE → REVOKED}
    Spanner-->>API: UpdatedCount
    API->>Redis: EvictMatchingTokens
  end

  API->>Audit: PublishRevokeEvent {selector_type, reason, revoked_count, domain, scope_qualifiers}
  API-->>Gateway: 200 {revoked_count, request_id}
  Gateway-->>Client: 200 RevokeResponse
```

---

## Failure And Control Notes

- **Detokenize fails closed**: Any policy check failure, policy service unavailability, or scope mismatch returns an error. No fallback to allow. MFA policy for human CDE access is enforced at the IdP layer, not bypassed by this service.
- **Scope mismatch on detokenize**: Returns `404 TOKEN_NOT_FOUND` — same response as token not found — to avoid disclosing whether the token exists under a different scope.
- **Cache failures**: Degrade to Spanner read path. Cache bypass is transparent to callers. Authorization cannot be bypassed on cache failure.
- **PFK and PEK are separate KMS operations**: The fingerprint computation (PFK) and PAN decryption (PEK) are distinct KMS calls with distinct keys and IAM roles. A compromise of one does not expose the other.
- **Audit events**: Every tokenize, detokenize, revoke, and denied request publishes to Pub/Sub. Audit events never contain PAN, CVV, or full key material.
- **PAN discovery (SR-08)**: Runs out-of-band against Cloud Logging buckets, BigQuery audit tables, and Spanner exports. Findings outside vault boundary trigger incident procedures per OR-04.
