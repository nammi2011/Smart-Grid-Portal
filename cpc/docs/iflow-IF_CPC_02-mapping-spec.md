# iFlow Mapping Specification — IF_CPC_02 `IF_CPC_Pref_SaveChanges`

**Version** 1.0 · **Status** For build · **Package** CPC (see `cpc-solution-package.md` §5)

The single write path for customer preferences. Carries the `saveChanges` payload from
the CPC UI (Service Cloud V2 mashup or customer portal) through API Management into the
S/4HANA RAP OData V4 action, and maps the result — including per-change errors — back to
the UI's response model.

> **[validate]** markers: OData V4 action URL pattern, principal-propagation setup, and
> JWT claim names must be confirmed against your landscape/IAS configuration.

---

## 1. Interface overview

| Attribute | Value |
|---|---|
| iFlow name | `IF_CPC_Pref_SaveChanges` |
| Direction | Inbound to S/4 (synchronous request/reply) |
| Sender | SAP API Management (which fronts: CPC UI in Service Cloud V2 mashup / customer portal) |
| Sender adapter | HTTPS, `POST /cpc/v1/preferences/saveChanges`, JSON |
| Receiver | S/4HANA RAP service `ZUI_CPC_PREF` (OData V4) |
| Receiver adapter | OData V4 (or HTTP with CSRF handling) → `POST <service-root>/saveChanges` **[validate exact action URL from service binding]** |
| QoS | Best effort, **no CPI retry** — user-driven transaction; the UI resubmits on failure. Idempotency via `requestId` (§4.1) |
| Payload limit | 256 KB; max **50** entries in `changes[]` (reject with `TOO_MANY_CHANGES`) |
| Timeout | 30 s end-to-end (APIM 32 s > CPI 30 s > OData receiver 25 s) |
| Security in | OAuth 2.0 bearer (IAS/XSUAA-issued JWT), validated at APIM, re-validated at CPI |
| Security out | Principal propagation to S/4 **[validate]**; fallback: OAuth2 technical user + user identity carried in payload fields set from JWT (§4.2) |
| Logging | MPL with correlation ID; payload logging **off** in PROD (PII); errors log partner + change count only, never contact addresses |

## 2. Processing steps (iFlow internals)

```mermaid
flowchart LR
    A[HTTPS sender\nPOST saveChanges] --> B[JSON schema validation\n(structure + enums)]
    B --> C[Enrich headers\ncorrelationId · requestId]
    C --> D[Identity guard\nJWT claims OVERRIDE payload\nchangedBy · role · source]
    D --> E[Semantic pre-checks\nscope/CA consistency · limits]
    E --> F[Message mapping\nUI JSON → ZCPC_A_SAVECHANGES]
    F --> G[Request-Reply\nOData V4 action]
    G --> H[Response mapping\nRAP result → UI response]
    H --> I[HTTPS response 200]
    B & D & E -.fail.-> X[Exception subprocess\nerror envelope §7]
    G -.OData error.-> X
```

Steps B, D, E fail fast **before** S/4 is called; step G errors are translated in §7.

## 3. Source message — UI `SaveChanges` JSON (as produced today, verified)

```json
{
  "requestId": "9f2a6c1e-4b7d-4e1a-9c33-0f8e2d5a7b91",
  "businessPartner": "0001000234",
  "contractAccount": "300012345",
  "source": "CSR",
  "changedBy": "kbutts",
  "changedByRole": "CSR",
  "onBehalfOf": "0001000234",
  "caseId": "8000123",
  "timestamp": "2026-08-17T18:11:35.248Z",
  "changes": [
    {
      "operation": "REPLACE",
      "preferenceType": "DNC",
      "scope": "BP",
      "contractAccount": null,
      "channel": "VOICE",
      "value": "OUT",
      "validFromNew": "2026-08-17",
      "endedRecordId": "PR-UMX5XL",
      "previousValue": "IN/VOICE",
      "reasonCode": "CUSTOMER_REQUEST"
    }
  ]
}
```

`requestId` is new (idempotency, §4.1): the UI adds one UUID per save attempt.

## 4. Header & identity handling

### 4.1 Headers

| Header (in) | Rule | Propagated as |
|---|---|---|
| `Authorization` | Validated (JWT); never forwarded to logs | Principal propagation / removed |
| `X-Correlation-ID` | Take if present, else generate UUID | OData header `X-Correlation-ID`; MPL custom header; returned in response |
| `X-Request-ID` | From payload `requestId` if header absent | Duplicate-detect: CPI data store `CPC_SAVE_REQ` (key = requestId, TTL 24 h). Duplicate → **replay stored response**, do not call S/4 |
| `Content-Type` | Must be `application/json` else `415` | `application/json` |

### 4.2 Identity guard — payload fields the iFlow OVERWRITES from the JWT

**Never trust client-supplied identity.** These payload fields are replaced before mapping:

| Payload field | Overwritten with | JWT source **[validate claim names]** |
|---|---|---|
| `changedBy` | Authenticated user ID | `sub` / `user_name` claim |
| `changedByRole` | Derived role | scope/role collection: `CPC.CSR` → `CSR`; `CPC.Customer` → `CUSTOMER` |
| `source` | Derived origin | OAuth client ID: SC2-mashup client → `CSR`; portal client → `PORTAL` (payload value ignored) |
| `onBehalfOf` | `businessPartner` when role=CSR, else cleared | — |
| `caseId` | Kept **only** when role=CSR, else cleared | — |
| `businessPartner` (CUSTOMER role only) | Partner mapped from subject via `ZCPC_USERMAP` — a customer can only ever save their own partner; mismatch → `403 FOREIGN_PARTNER` | — |

## 5. Field mapping — request (UI JSON → `ZCPC_A_SAVECHANGES`)

Target: parameter structure of static action `saveChanges` (RAP, OData V4).

### 5.1 Header fields

| # | Source (JSON path) | Target (ABAP field) | Type/len | M | Transformation |
|---|---|---|---|---|---|
| 1 | `requestId` | `REQUEST_ID` | CHAR 36 | M | Pass-through; UUID format check |
| 2 | `businessPartner` | `PARTNER` | `BU_PARTNER` C10 | M | **ALPHA input conversion** (left-pad zeros to 10); after identity guard §4.2 |
| 3 | `contractAccount` | `VKONT` | `VKONT_KK` C12 | O | ALPHA to 12; `null` → initial |
| 4 | `source` (guarded) | `SOURCE` | C12 | M | Value map VM-03 |
| 5 | `changedBy` (guarded) | `CHANGED_BY` | C40 | M | Uppercase not applied (keep as issued) |
| 6 | `changedByRole` (guarded) | `CHANGED_BY_ROLE` | C10 | M | `CSR` / `CUSTOMER` only |
| 7 | `onBehalfOf` (guarded) | `ON_BEHALF_OF` | `BU_PARTNER` C10 | O | ALPHA; only when role=CSR |
| 8 | `caseId` (guarded) | `CASE_ID` | C20 | O | Only when role=CSR |
| 9 | `timestamp` | `CLIENT_TS` | `TIMESTAMPL` | O | ISO 8601 → UTC `YYYYMMDDhhmmss.mmmmmmm`; informational only — S/4 stamps its own `CHANGED_AT` |
| 10 | header `X-Correlation-ID` | `CORRELATION_ID` | C40 | M | From §4.1 |

### 5.2 `changes[]` line items (1..50) → `CHANGES` table parameter

| # | Source | Target | Type/len | M | Transformation |
|---|---|---|---|---|---|
| 11 | `operation` | `OPERATION` | C8 | M | Value map VM-01 (`SET`/`REPLACE`/`END`) |
| 12 | `preferenceType` | `PREF_TYPE` | C20 | M | Uppercase; must exist in `ZCPC_PREFTYPE` (S/4 validates; iFlow only checks charset `[A-Z0-9_]{1,20}`) |
| 13 | `scope` | `SCOPE` | C2 | M | VM-02 (`BP`/`CA`) |
| 14 | `contractAccount` | `VKONT` | C12 | C | **Consistency rule R1**: `scope=BP` → must be `null` → initial; `scope=CA` → mandatory, ALPHA-padded. Violation → fail fast `SCOPE_CA_MISMATCH` (no S/4 call) |
| 15 | `channel` | `CHANNEL` | C10 | M | VM-04 (`EMAIL/SMS/VOICE/PUSH/POST/ALL`) |
| 16 | `value` | `PREF_VALUE` | C3 | M | `IN`/`OUT` only |
| 17 | `validFromNew` | `VALID_FROM` | `DATS` | C | ISO `yyyy-MM-dd` → `yyyyMMdd`; mandatory for SET/REPLACE; **not in the past** beyond 30 days, not > 1 year future (else `INVALID_VALID_FROM`); absent for END |
| 18 | `endedRecordId` | `ENDED_PREF_ID` | C36 | C | Mandatory for REPLACE/END; UI record key `PR-*` OR UUID — pass through; S/4 resolves demo-style IDs only in non-PROD **[design note: production UI sends the real `PreferenceUUID` it read]** |
| 19 | `previousValue` | `PREV_VALUE` | C30 | O | Informational (concurrency display); authoritative previous value re-read in S/4 |
| 20 | `reasonCode` | `REASON_CODE` | C20 | O | VM-05; default `CUSTOMER_REQUEST` |

### 5.3 Value mappings

**VM-01 `operation`** — identical values, mapping table exists for future variants:
`SET→SET`, `REPLACE→REPLACE`, `END→END`; anything else → fail `INVALID_OPERATION`.

**VM-02 `scope`**: `BP→BP`, `CA→CA`; default when absent: derive from `ZCPC_PREFTYPE`
lookup is **not** done in CPI — absent scope → fail `MISSING_SCOPE` (UI always sends it).

**VM-03 `source`** (after identity guard): `CSR→CSR`, `PORTAL→PORTAL`; migration/OMS/DRMS
loads use IF-specific clients mapping to `MIGRATION`/`OMS`/`DRMS` (separate iFlow configs
of the same template).

**VM-04 `channel`**: pass-through on the six values; lowercase input tolerated (uppercased).

**VM-05 `reasonCode` allowlist**: `CUSTOMER_REQUEST`, `CSR_CORRECTION`, `COMPLIANCE`,
`MIGRATION`, `BRF_AUTO` (last two rejected unless source matches).

## 6. Field mapping — response (RAP result → UI JSON)

RAP action returns `ZCPC_A_SAVERESULT`: header + per-change results + refreshed active set.

| # | Source (ABAP) | Target (JSON) | Notes |
|---|---|---|---|
| 1 | `REQUEST_ID` | `requestId` | Echo |
| 2 | `STATUS` (`S`/`E`/`P`) | `status` | `S→"success"`, `P→"partial"`, `E→"error"` |
| 3 | `SAVED_COUNT` | `savedCount` | |
| 4 | `RESULTS[]`: `LINE_NO`, `PREF_TYPE`, `STATUS`, `NEW_PREF_ID`, `MSG_CODE`, `MSG_TEXT` | `results[]`: `index`, `preferenceType`, `status`, `newRecordId`, `errorCode`, `message` | `MSG_TEXT` passed only for E-lines; customer-safe (RAP messages authored accordingly) |
| 5 | `ACTIVE_SET[]` (projection of `ZC_CPC_CustomerPref`, active rows) | `preferences[]` | Field names camelCased 1:1 (PreferenceUUID→`id`, PreferenceType→`type`, …) so the UI re-renders without a second read |
| 6 | `CORRELATION_ID` | header `X-Correlation-ID` + `correlationId` | |

**Partial saves (`status=partial`)**: the RAP action processes `changes[]`
**all-or-nothing per line but not across lines** — each line commits independently?
**No — design decision D1: the whole `changes[]` is ONE LUW.** Any line error rolls back
all lines; `results[]` then marks every line (`ok-but-rolled-back` lines get status
`ROLLED_BACK`). Rationale: a half-applied preference set (e.g., end DNC succeeded, create
replacement failed) is a compliance hazard. `P/partial` therefore never occurs in v1.0
and is reserved.

## 7. Error handling & mapping

Exception subprocess maps every failure to the standard envelope; HTTP code chosen per class:

```json
{ "error": { "code": "SCOPE_CA_MISMATCH",
             "message": "We couldn't save your preferences. Please refresh and try again.",
             "correlationId": "…", "retryable": false,
             "details": [ { "index": 2, "errorCode": "SCOPE_CA_MISMATCH",
                            "message": "BP-level preference must not carry a contract account." } ] } }
```

| HTTP | `error.code` | Raised by | Condition | Retryable |
|---|---|---|---|---|
| 400 | `SCHEMA_INVALID` | CPI step B | JSON malformed / enum invalid / >50 changes (`TOO_MANY_CHANGES`) | no |
| 400 | `SCOPE_CA_MISMATCH`, `MISSING_SCOPE`, `INVALID_VALID_FROM`, `INVALID_OPERATION` | CPI step E | Rules §5.2 | no |
| 401 | `UNAUTHENTICATED` | APIM/CPI | JWT missing/expired | no (re-login) |
| 403 | `FOREIGN_PARTNER` | CPI step D / S/4 DCL | Customer role, partner ≠ own | no |
| 404 | `PARTNER_NOT_FOUND`, `RECORD_NOT_FOUND` | S/4 | Unknown BP / `ENDED_PREF_ID` not found or already ended | no |
| 409 | `CONCURRENT_CHANGE` | S/4 (etag on `ChangedAt`) | Record changed since UI read | no — UI reloads and reapplies |
| 409 | `DUPLICATE_REQUEST` | CPI §4.1 | Same `requestId` seen — stored response replayed with this code only if original still processing | no |
| 422 | `MANDATORY_OPTOUT` | BRF `FN_VALIDATE_CHANGE` | Opt-out attempt on mandatory type (PAST_DUE) | no |
| 422 | `CHANNEL_NOT_ALLOWED` | BRF | Channel not in type's allowed list | no |
| 502 | `BACKEND_ERROR` | CPI step G | OData 5xx / mapping failure of S/4 response | yes |
| 503 | `BACKEND_UNAVAILABLE` | CPI step G | Timeout / connection refused | yes |

Customer-safe `message` texts are fixed per code (UI shows them verbatim); technical
detail goes to MPL + `details[]` only. 5xx responses include `Retry-After: 30`.

## 8. Idempotency & concurrency summary

- **Idempotency**: `requestId` dedup store (24 h TTL) in CPI; replay returns the stored
  response body — safe for UI auto-retry on 502/503.
- **Concurrency**: optimistic — `ENDED_PREF_ID` + etag(`ChangedAt`); loser gets 409 and
  the UI reloads the active set (returned in every success response, §6.5).
- **Ordering**: single LUW per request (D1); cross-request ordering per partner is not
  guaranteed and not required (last-write-wins with full audit).

## 9. Test cases (build acceptance for this iFlow)

| # | Case | Expected |
|---|---|---|
| M1 | Valid CSR save (mashup JWT), 3 changes | 200, `status=success`, 3 audit rows in S/4, `preferences[]` returned; MPL has correlation ID |
| M2 | Payload claims `changedBy=admin` but JWT user is `kbutts` | Persisted `CHANGED_BY=kbutts` (identity guard) |
| M3 | Customer JWT, foreign `businessPartner` | 403 `FOREIGN_PARTNER`; S/4 never called |
| M4 | `scope=BP` with `contractAccount` filled | 400 `SCOPE_CA_MISMATCH`; S/4 never called |
| M5 | Opt-out of `PAST_DUE` | 422 `MANDATORY_OPTOUT` from BRF, envelope `details[]` names the line |
| M6 | Same `requestId` sent twice | Second call returns identical stored response; exactly one set of audit rows |
| M7 | S/4 down | 503 `BACKEND_UNAVAILABLE`, `retryable=true`, `Retry-After` |
| M8 | One bad line among 5 | 4xx with all lines rolled back (D1); zero rows written |
| M9 | 51 changes | 400 `TOO_MANY_CHANGES` |
| M10 | DNC change end-to-end | Save OK **and** `zcpc/DncChanged` observed on Event Mesh (ENH-05) |

## 10. Open items for build

1. **[validate]** exact OData V4 action URL + CSRF handling from the published binding.
2. **[validate]** principal propagation vs technical-user fallback (affects §4.2 rows 1–3).
3. **[validate]** JWT claim names for user/role in your IAS setup.
4. ~~UI change (small): add `requestId` UUID per save attempt (§3)~~ — **DONE** (deployed):
   UUID created on first staged change, stable across renders, reused on failed-save
   retry, rotated after success, cleared on discard.
5. Confirm D1 (single LUW, no partial saves) with the business — spec assumes yes.
