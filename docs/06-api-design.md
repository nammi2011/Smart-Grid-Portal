# Deliverable 13 — API Design (REST) — see `openapi.yaml` for the machine-readable spec

Base path: `/api/v1` (approuter route → CAP service). All endpoints require a Bearer
JWT; customers are constrained to their own resources (instance-based authorization —
see security doc). Errors use a uniform envelope.

## Error envelope & status codes

```json
{
  "error": {
    "code": "USAGE_UNAVAILABLE",
    "message": "We're unable to retrieve your usage information right now.",
    "correlationId": "b7c1a9e2-...",
    "retryable": true
  }
}
```

| Status | Use |
|---|---|
| 200/201 | Success / created |
| 400 | Validation failure (`details[]` lists field errors) |
| 401 / 403 | Missing/invalid token / not the resource owner or missing role |
| 404 | Resource not found (or not visible to caller — indistinguishable by design) |
| 409 | Conflict (e.g., duplicate active enrollment) |
| 422 | Business rule rejection (e.g., not eligible) |
| 429 | Throttled (API Management policy) |
| 503 | Downstream dependency unavailable (retryable=true) |

## Endpoints

| Method & path | Purpose | Notes |
|---|---|---|
| `GET /customers/{customerId}` | Customer profile + attributes (EV, solar, meter) | owner or CSR |
| `GET /accounts/{accountId}` | Account, balance, next payment, current rate | |
| `GET /accounts/{accountId}/usage?months=12&resolution=monthly` | Monthly TOU aggregates; `resolution=interval&date=` for hourly | interval fetch proxies MDMS w/ cache |
| `GET /accounts/{accountId}/preferences` | All preferences grouped by section | |
| `PUT /accounts/{accountId}/preferences` | Bulk upsert changed preferences; writes audit; emits event | body: `{changes:[{category,key,value,channel?,threshold?}]}` |
| `GET /accounts/{accountId}/consents` / `PUT .../consents` | Consent read/update + history | |
| `GET /programs?category=&filter=eligible|recommended|enrolled` | Marketplace catalog | filter applies eligibility engine |
| `GET /programs/{programId}` | Program detail incl. terms | |
| `POST /programs/{programId}/eligibility-check` | Evaluate rules for caller's account | returns outcome + per-rule reasons |
| `POST /programs/{programId}/enroll` | Create enrollment (Submitted) | 422 if not eligible; 409 if duplicate |
| `GET /enrollments` / `DELETE /enrollments/{id}` | My programs / cancel where allowed | |
| `GET /rates` / `GET /rates/eligible` | Rate catalog / rates the caller may switch to | |
| `POST /rates/analyze` | Run analysis for a period | body: `{accountId, months:12|24, from?, to?}` |
| `POST /rates/simulate` | What-if simulation | body: scenario adjustments (below) |
| `POST /rates/change-request` | Request switch to eligible rate | re-verifies eligibility server-side |
| `GET /rate-change-requests` | Status of my requests | |
| `GET /notifications` / `PUT /notifications/{id}/read` | Notification center | |

## Representative request/response

`POST /rates/analyze`
```json
{ "accountId": "1000456789", "months": 12 }
```
```json
{
  "analysisId": "…",
  "period": { "from": "2025-08", "to": "2026-07", "months": 12 },
  "currentRate": { "id": "R1", "name": "Residential Standard R1",
                   "annualCost": 2119.44, "monthlyAverage": 176.62 },
  "alternatives": [
    { "id": "RT-EV", "name": "EV Time-of-Use", "annualCost": 1988.70,
      "savings": 130.74, "savingsPercent": 6.2, "eligible": true, "risk": "MEDIUM" }
  ],
  "recommendation": {
    "rateId": "RT-EV", "confidence": 0.87,
    "explanation": "About 62% of your electricity usage occurs during off-peak and overnight hours…"
  }
}
```

`POST /rates/simulate`
```json
{
  "accountId": "1000456789",
  "baseMonths": 12,
  "scenario": {
    "evChargingWindow": "OVERNIGHT",     // EVENING | OVERNIGHT
    "evMonthlyKwh": 300,
    "thermostatSetback": true,            // shifts 8% of peak kWh to off-peak
    "peakShiftPercent": 0,                // additional manual peak→off-peak shift
    "solarKw": 0,                         // reduces daytime kWh
    "battery": false,                     // shifts up to 150 kWh/mo peak→super-off-peak
    "consumptionGrowthPercent": 0
  }
}
```
Response: adjusted usage profile + per-rate monthly/annual costs + best plan + savings
vs. current — all computed by the rate engine (no hard-coded results).

**Validation:** JSON-schema per endpoint (OpenAPI), CAP input validation annotations,
threshold ranges (e.g., monthly bill alert $10–$5,000), enum checks; unknown fields
rejected.
