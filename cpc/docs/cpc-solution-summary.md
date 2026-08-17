# Customer Preference Center — Solution Summary

**One platform for how a utility talks to its customers.** Customers manage their own
communication preferences on the portal; CSRs manage them inside SAP Service Cloud V2;
the rules and data live in SAP S/4HANA Utilities; and every outbound message — bill,
outage alert, reminder, offer — is checked against those preferences at the moment it
is sent.

## What it does

- **Preference management** — communication channels, billing, usage and outage alerts,
  program and marketing opt-ins, privacy consents. Records are validity-dated and
  end-dated, never deleted: the full history is audit-ready by construction.
- **Do-Not-Call done right** — DNC/Do-Not-Mail held at Business Partner level, applying
  to every contract account, distributed to dialers, campaign tools and print with
  delivery receipts, sequence protection and daily reconciliation.
- **Event-driven communication** — business events (bill created, outage detected,
  demand-response dispatch…) resolve *whether and how* to contact the customer through
  BRFplus **at send time**; suppressed messages are logged with their reason as
  compliance evidence.
- **Automatic housekeeping** — Move-In/Move-Out and BP changes trigger BRFplus rules
  that end, seed or revalidate preferences with no manual CSR effort.

## How it is built

| Layer | Component |
|---|---|
| Front ends | One embeddable web app: **Service Cloud V2 mashup** (CSR, case-context) and customer portal mode. *Option B:* on-stack S/4 CRM WebClient assignment block |
| Integration | SAP BTP: API Management + **Integration Suite** (8 iFlows — save, DNC broadcast, dispatch built as importable CPI projects) + Event Mesh |
| Logic | **BRFplus application ZCPC** — validation, channel resolution with DNC fallback, event auto-processing; rules maintained by business admins |
| Data & API | S/4 tables (`ZCPC_PREF/AUDIT/PREFTYPE`) + CDS views + **RAP OData V4 service**; one core class is the only writer for both front ends |
| Security | Identity from the authenticated token — never the payload; customers restricted to their own partner; insert-only audit with CSR on-behalf-of attribution; correlation IDs end to end |

**Design principles:** one write path (`ZCL_CPC_CORE`) · half-open validity
`[from, to)` · BP-level privacy preferences · all-or-nothing saves · send-time
decisions, never cached · no invented SAP APIs — every landscape-dependent artifact is
flagged **[validate]**.

## What exists today

| Asset | Status |
|---|---|
| Live demo (4 mock customers, mashup-ready) | **Running** — `nammi2011.github.io/Smart-Grid-Portal/cpc/` |
| Service Cloud V2 mashup configuration | Documented & rehearsed |
| CPI iFlows (save · DNC broadcast · dispatch) | Built; **IF_CPC_02 imported successfully** |
| S/4 build kit: table DDL, CDS views, RAP BDEF + behavior pool, core class, GenIL adapter | Delivered (code, pre-syntax-check) |
| BRFplus: architecture, step-by-step build guide, seed content (24 types, DNC policy, event rules) | Delivered |
| Full documentation package + client deck | Delivered (single combined Word document) |

## Delivery at a glance

**Phase 0 (~1 wk)** validate landscape items, confirm option → **Phase 1 (6–8 wks)** S/4
core: tables, CDS, RAP, BRFplus, event enhancements → **Phase 2 (3–4 wks)** BTP deploy,
Event Mesh, Service Cloud rollout → **Phase 3 (3–4 wks)** migration & cutover.
**Total ≈ 13–18 weeks** (Option A). Option B adds ~3 weeks of WebClient UI.

**Why SAP BTP:** the system of record is S/4HANA Utilities and the agent desktop is
Service Cloud V2 — on BTP, connectivity, identity, integration runtime, eventing and
monitoring are subscribed services and sanctioned extension patterns; on a custom
stack each is a build-and-maintain project before the first preference is saved.

*All customer data shown in demos is fictional. Estimates are indicative pending Phase 0 validation.*
