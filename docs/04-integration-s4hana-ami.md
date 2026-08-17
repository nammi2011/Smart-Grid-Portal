# Deliverable 9–11 — Integration Architecture, S/4HANA Utilities & AMI/MDMS Integration

> **Landscape caveat (applies to every row below):** exact OData/SOAP services, events,
> and extensibility points differ by S/4HANA Utilities release, deployment model
> (on-premise / private cloud), and enabled business scope. Every named API is
> **"To be validated against target SAP S/4HANA Utilities landscape."** The
> integration *requirement* and pattern stand regardless.

## 1. Integration architecture

```mermaid
flowchart LR
    P[Portal CAP Services] --> DEST[BTP Destination Service]
    DEST --> APIM[Integration Suite\nAPI Management\n(policies: quota, spike arrest, JWT)]
    APIM --> CI[Cloud Integration iFlows\nmapping · retry · error handling]
    CI --> S4[S/4HANA Utilities]
    CI --> MDMS[MDMS / AMI Head-End]
    CI --> DRMS[DRMS / Program Mgmt]
    CI --> OMS[OMS]
    CI --> MKT[Marketing Platform]
    P --> EM[[Event Mesh]] --> CI
    S4 -- business events --> EM
```

Patterns used: **synchronous API** (profile, balances, rates, on-demand usage),
**async event** (preference sync, enrollment provisioning, rate change), **batch/
replication** (nightly aggregated usage to the analytics cache).

## 2. Conceptual S/4HANA Utilities object mapping

| Portal concept | S/4HANA Utilities object | Notes |
|---|---|---|
| Customer | Business Partner | BP ID is the cross-system key |
| Account | Contract Account (FI-CA) | Balance, due date, payment data |
| Utility Contract | Utility Contract (move-in/out status) | Supply relationship |
| Premise / Service address | Premise + Connection Object | |
| Installation | Installation | Links premise ↔ rate ↔ devices |
| Meter | Device / Equipment (Device Management) | AMI capability flag |
| Rate Plan | Rate Category / Rate (IS-U Billing) | Portal keeps a presentation model; S/4 remains tariff master |
| Billing/consumption history | Billing documents / meter reading results | |
| Move-in/Move-out status | Utility Contract processes | Read-only in portal MVP |

## 3. Integration mapping table

| Portal function | Source | Target | Pattern | API / mechanism | Frequency |
|---|---|---|---|---|---|
| Customer profile & accounts | S/4HANA Utilities | Portal | Sync API via Destination + Integration Suite | Business Partner & Contract Account read APIs — *to be validated per landscape* | Real-time, cached 15 min |
| Account balance / next payment | S/4HANA Utilities (FI-CA) | Portal | Sync API | Contract Account balance API — *to be validated* | Real-time |
| Customer preferences | Portal (system of record) | CRM / Marketing / OMS / DRMS | Async event + iFlow fan-out | `CustomerPreferenceChanged` via Event Mesh | Near real-time |
| Consent records | Portal | Enterprise consent repository / marketing | Async event | `CustomerConsentChanged` | Near real-time |
| Program catalog | DRMS / Program Mgmt | Portal | Batch sync or admin-maintained in portal | REST pull, nightly | Daily |
| Program enrollment | Portal | DRMS / Program Mgmt | Async event + callback | `ProgramEnrollmentSubmitted` → provisioning → status callback | Near real-time |
| Meter usage (monthly) | S/4 billing / MDMS | Portal cache | Batch replication | Aggregates to HANA Cloud cache | Nightly |
| Interval usage | MDMS | Portal | Sync API on demand (cached) | MDMS REST (vendor-specific) | On demand |
| Rate master & pricing components | S/4HANA Utilities / rate engine | Portal rate model | Admin-governed sync | Export/replication — *to be validated* | On rate change |
| Rate change request | Portal | S/4HANA Utilities | Async workflow + iFlow | Contract/rate category change — *to be validated* | Per request |
| Outage notifications | OMS | Portal / notification channels | Event | OMS event feed → Event Mesh | Real-time |
| Payments (link-out) | Portal | Payment gateway | Redirect / hosted page | Provider SDK | Real-time |

## 4. AMI / MDMS integration design

```
Smart Meter → Head-End System (HES) → MDMS → Integration Suite / API layer → Rate Analyzer
```

**Do not store raw AMI reads in the portal transactional DB.** A single meter at
15-minute cadence produces ~35k intervals/year; thousands of customers make raw
storage in the portal DB unnecessary and expensive. Strategy:

1. **Aggregate at the edge**: Integration Suite (or MDMS export) produces
   *monthly TOU aggregates* per contract: kWh by period (peak / off-peak / super
   off-peak), peak demand kW, billed amount → stored in HANA Cloud
   (`ConsumptionHistory`). This is all the rate engine needs for 12–24-month analysis.
2. **On-demand interval retrieval**: daily/hourly detail fetched from MDMS API only
   when the customer opens daily views or runs high-resolution simulation; cached in
   HANA (or Redis-style cache) with 24 h TTL (`IntervalUsage` holds representative
   profiles + cache entries, not the raw archive).
3. **Analytics storage (Phase 2+)**: full interval history belongs in the analytics
   stack (HANA Cloud data lake / Datasphere), feeding ML rate recommendations —
   separate from the transactional schema.
4. **Retention (proposed)**: portal cache — 24 months aggregates, 60 days interval
   cache; analytics store — 36+ months per regulatory requirements; raw reads remain
   in MDMS per its own retention policy.

## 5. Failure handling per integration

- Every outbound call carries a **correlation ID** (propagated from the portal request).
- Sync calls: 2 retries with backoff at the iFlow level, circuit-breaker in API
  Management; portal falls back to cached data with a "data may be delayed" banner.
- Async events: Event Mesh redelivery with exponential backoff; after max attempts →
  dead-letter queue; Alert Notification raises an ops incident; portal status remains
  "Submitted/Under Review" until the callback arrives (never silently "Enrolled").
