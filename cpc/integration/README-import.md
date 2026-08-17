# CPC iFlows — CPI Import Guide

Three importable integration-flow projects:

| ZIP | Flow | Endpoint |
|---|---|---|
| `IF_CPC_02_Pref_SaveChanges.zip` | Preference save (portal/SC2 → S/4) | `POST /http/cpc/v1/preferences/saveChanges` |
| `IF_CPC_06_DNC_Broadcast.zip` | DNC/DNM distribution to dialer, campaign, print + audit receipts | `POST /http/cpc/v1/events/dnc` (Event Mesh **webhook**) |
| `IF_CPC_07_Comm_Dispatch.zip` | Event-driven communication: resolveComm at send time → provider → audit | `POST /http/cpc/v1/events/comm` (Event Mesh **webhook**) |

**v1 design choices for import reliability (documented deviations from the specs):**

1. **Webhook, not AMQP** — 06/07 consume events via Event Mesh *webhook subscriptions*
   pushing to their HTTPS endpoints. Retry = webhook redelivery driven by the response
   code (200 done · 4xx park · 5xx/Retry-After redeliver). Swap to an AMQP sender in
   the editor later if queue-based consumption is preferred.
2. **Sequential targets with no-throw HTTP** (06) — dialer → campaign → print are
   called in sequence with `throwExceptionOnFailure=false`; each outcome is recorded
   per target and a partial failure returns 500 for redelivery. Targets must be
   idempotent on `externalRef` = event id (per spec) since confirmed targets may be
   re-called on redelivery. The spec's parallel-branch model can be restored with a
   multicast in the editor.
3. **Provider gateway** (07) — one HTTP receiver (`PROVIDER_GATEWAY_URL`) receives
   `{channel, to, templateId, variables, clientRef, priority}`; per-channel adapters
   can replace it later with a channel router.
4. **Quiet hours / resolve-down** (07) — deferrals answer `503 + Retry-After` so the
   webhook redelivers; the decision is **re-resolved on every attempt** (rule R0).
   Quiet-hours use server-local time in v1 (recipient-timezone is a Phase-2 item).

Import for all three: Design → package → Add → Integration Flow → **Upload** →
**Configure** parameters → **Deploy**. Data stores (`CPC_SAVE_REQ`, `CPC_DNC_EVT`,
`CPC_DNC_SEQ`, `CPC_COMM_EVT`) are created implicitly on first write.

---

# IF_CPC_02 — details

`IF_CPC_02_Pref_SaveChanges.zip` is a Cloud Integration flow project in the standard
exported-iFlow format. Implements the IF_CPC_02 mapping spec v1.0: HTTPS sender →
**Validate + Identity Guard** (Groovy) → **Call S/4 saveChanges** (HTTP request-reply)
→ **Map Response + replay store** (Groovy), with an exception subprocess building the
customer-safe error envelope (incl. idempotent replay on duplicate `requestId`).

> **Honesty note:** this ZIP was authored to the exported-project structure but has
> **not been imported into a live CPI tenant** from here. If your tenant rejects it or
> renders oddly, the three Groovy scripts inside carry 100% of the logic — recreate the
> 5-step flow in the web editor and paste them in (≈15 min). Report the exact import
> error and it will be fixed.

## Import

1. Integration Suite → **Design** → your package → **Add → Integration Flow →
   Upload** → select the ZIP.
2. Open the flow → **Configure** (externalized parameters):

| Parameter | Set to |
|---|---|
| `S4_SAVECHANGES_URL` | Your RAP action URL from the `ZUI_CPC_PREF` OData V4 binding **[validate exact URL]** |
| `S4_AUTH_METHOD` / `S4_CREDENTIAL_NAME` | `OAuth2ClientCredentials` + your credential artifact name (create under Security Material first) |
| `S4_PROXY_TYPE` | `default` (internet) or `onPremise` (Cloud Connector) |
| `S4_TIMEOUT_MS` | 25000 |
| `CPC_MAX_CHANGES` / `CPC_DEDUP_ENABLED` | 50 / true |

3. **Deploy**. Endpoint: `https://<runtime-host>/http/cpc/v1/preferences/saveChanges`
   (sender auth: role `ESBMessaging.send` — call with a client certificate/OAuth client
   that has it; API Management sits in front in the target architecture).

## Contract with API Management (identity guard)

The flow **requires** these verified headers from APIM policies (it rejects requests
without them — never trusting payload identity):

| Header | Content |
|---|---|
| `X-Verified-User` | Authenticated user ID (from validated JWT) |
| `X-Verified-Role` | `CSR` or `CUSTOMER` |
| `X-Verified-Partner` | The customer's own BP (required for role CUSTOMER; enforces own-partner-only) |
| `X-Verified-Client` | OAuth client ID (informational) |
| `X-Correlation-ID` | Optional; generated if absent |

## Smoke test (after deploy)

```bash
curl -X POST https://<runtime>/http/cpc/v1/preferences/saveChanges \
  -H "Content-Type: application/json" \
  -H "X-Verified-User: kbutts" -H "X-Verified-Role: CSR" \
  -d '{"requestId":"9f2a6c1e-4b7d-4e1a-9c33-0f8e2d5a7b91","businessPartner":"0001000234",
       "contractAccount":"300012345","changes":[{"operation":"SET","preferenceType":"PAPERLESS",
       "scope":"CA","contractAccount":"300012345","channel":"EMAIL","value":"IN",
       "validFromNew":"2026-08-17"}]}'
```

Expected without S/4 connected: `503 BACKEND_UNAVAILABLE` envelope (validation passed,
backend absent). Bad payload (e.g. `scope:"BP"` with a contract account) → `400
SCOPE_CA_MISMATCH` **without any S/4 call**. Duplicate `requestId` after one success →
replayed stored response. Create the Data Store name `CPC_SAVE_REQ` implicitly on first
write (no setup needed).

## Contents

```
META-INF/MANIFEST.MF                    bundle manifest (IntegrationFlow)
src/main/resources/
  scenarioflows/integrationflow/IF_CPC_02_Pref_SaveChanges.iflw   BPMN2 flow
  script/ValidateAndGuard.groovy        spec §2 B–E, §4, §5 (validation, guard, dedup)
  script/MapResponse.groovy             spec §6 (+ replay store, 24 h)
  script/ErrorEnvelope.groovy           spec §7 (error catalog incl. 409/422 mapping)
  parameters.prop / parameters.propdef  externalized parameters
```
