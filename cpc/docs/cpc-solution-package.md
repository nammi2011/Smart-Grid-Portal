# Customer Preference Center — Complete Solution Package

**Scope.** End-to-end design and configuration for the Customer Preference Center (CPC):
preferences maintained by customers and CSRs, logic and event processing in **SAP S/4HANA
Utilities BRFplus**, data persisted in S/4 (tables + **CDS views** + RAP OData service),
**Integration Suite iFlows** for create/update and event distribution, and enhancements
that act on business events. Two front-end options are specified:

- **Option A** — application UI resides on **SAP BTP**, embedded in **SAP Service Cloud V2**
  (CSR) and the customer portal (self-service).
- **Option B** — front end is **on-stack S/4HANA CRM** (SAP S/4HANA for Customer
  Management / Interaction Center), matching the original CPC01–CPC19 specification set.

> **Terminology note.** CDS views *expose* data; storage is the underlying tables
> (`ZCPC_PREF`, `ZCPC_AUDIT`, `ZCPC_PREFTYPE`). "CDS views to store master data and
> preferences" is implemented as: tables → CDS interface views → consumption view →
> RAP OData V4 service. DDL for all of it is delivered in `s4-artifacts/` (files 01–03).
>
> **Landscape caveat.** Standard SAP artifacts named here (FDT classes, event topics,
> `I_BusinessPartner`, RAP business events, S/4HANA for Customer Management) are real but
> release/scope-dependent — each is marked **[validate]** and must be confirmed against
> your landscape. All `Z*` artifacts are the custom developments this package specifies.

---

## 1. Package contents

| # | Deliverable | File |
|---|---|---|
| 1 | This solution document (Options A + B, configuration, iFlows, enhancements) | `docs/cpc-solution-package.md` (+ .docx) |
| 2 | BRFplus architecture & API design | `docs/cpc-brfplus-architecture.md` (+ .docx) |
| 3 | BRFplus decision-table seed content (functional review workbook) | `docs/CPC-BRFplus-DecisionTable-Content.xlsx` |
| 4 | Table DDL (`ZCPC_PREFTYPE`, `ZCPC_PREF`, `ZCPC_AUDIT`) | `s4-artifacts/01_zcpc_tables.txt` |
| 5 | CDS interface views + DCL sketch | `s4-artifacts/02_zi_cpc_views.asddls.txt` |
| 6 | Consumption view, RAP behavior sketch, service definition | `s4-artifacts/03_zc_cpc_service.asddls.txt` |
| 7 | Working CPC UI (embeddable; 4 mock customers) | `index.html` — live: `https://nammi2011.github.io/Smart-Grid-Portal/cpc/` |
| 8 | Production hosting config (nginx `/cpc/`, frame-ancestors) | `customer-portal/nginx/svgbs.com.conf` |

---

## 2. Option comparison and recommendation

| Criterion | **Option A — BTP app + Service Cloud V2** | **Option B — S/4HANA CRM (on-stack)** |
|---|---|---|
| CSR front end | CPC web app embedded as Service Cloud V2 **URL mashup** on the Case/Account screen | CPC **assignment block** on BP Overview / Interaction Center (WebClient UI), per legacy CPC01 spec |
| Customer self-service | Same app in `mode=customer` on the portal — one codebase | Requires a separate portal UI anyway (IC WebClient is agent-only) |
| Where logic runs | BRFplus in S/4 (identical in both options) | BRFplus in S/4 (identical) |
| Where data lives | S/4 tables + CDS/RAP (identical) | S/4 tables + CDS/RAP (identical) |
| Integration need | BTP ↔ S/4 via Integration Suite (iFlows below) + Cloud Connector | Minimal — UI and data in one stack; iFlows only for outbound distribution |
| UI technology risk | Low — standard web app, already built and tested | **S/4HANA for Customer Management availability/roadmap must be validated** [validate]; WebClient UI skills increasingly rare |
| Fit with Service Cloud V2 strategy | Native (mashup already configured/tested) | Service Cloud V2 not in the loop; CSRs work in two UIs |
| Delivery effort (indicative) | UI exists; effort concentrates in S/4 API + iFlows | UI development on WebClient framework is the largest work item |
| **Recommendation** | **Preferred** if Service Cloud V2 is the strategic agent desktop | Choose only if the agent desktop remains on-stack CRM/IC for the planning horizon |

Both options share **the entire S/4 core** (sections 4–6): tables, CDS, RAP service,
BRFplus, events, dispatcher. The option choice only swaps the presentation layer, so a
later migration B→A (or A→B) does not touch data or logic.

---

## 3. Option A — architecture

```mermaid
flowchart LR
    subgraph FE[Front ends]
      SC2[Service Cloud V2\nCase screen - URL mashup]
      PORT[Customer portal\nsvgbs.com/cpc]
    end
    subgraph BTP[SAP BTP]
      HTML5[CPC UI\nHTML5 repo / static host]
      APIM[API Management\nquota - JWT - spike arrest]
      CPI[Integration Suite\niFlows IF_CPC_*]
      EM[Event Mesh]
      DEST[Destination + Connectivity\n(Cloud Connector to S/4)]
      IAS[IAS / XSUAA]
    end
    subgraph S4[SAP S/4HANA Utilities]
      RAP[ZUI_CPC_PREF\nRAP OData V4]
      TAB[(ZCPC_PREF / AUDIT / PREFTYPE\n+ CDS views)]
      BRF[BRFplus ZCPC\nvalidate - resolve - auto-process]
      ENH[Enhancements\nmove-in/out - BP change - dispatch]
    end
    SC2 -->|iframe + params| HTML5
    PORT --> HTML5
    HTML5 -->|OAuth| APIM --> CPI
    CPI -->|OData| RAP
    RAP --> TAB
    RAP --> BRF
    ENH --> BRF
    ENH --> TAB
    RAP -. business events .-> EM
    EM --> CPI
    CPI --> OUT[Outbound providers\nEmail / SMS / Voice / Print]
    CPI --> DS[Downstream: OMS - DRMS - Marketing - Dialer]
    IAS -.auth.- HTML5
```

**Component responsibilities**

| Layer | Component | Responsibility |
|---|---|---|
| UI | CPC app (delivered, `index.html`) | Active/history view, end-dating, DNC banner, BRF+ previews; `mode=csr|customer`; emits the `SaveChanges` payload |
| BTP | API Management | Public API facade: JWT validation, rate limits, customer-safe errors, correlation ID injection |
| BTP | Integration Suite | iFlows (sec. 5) — the only path between BTP and S/4 |
| BTP | Event Mesh | `zcpc/*` topics; queues per consumer with DLQ |
| BTP | Destination + Cloud Connector | Technical connectivity + principal propagation to S/4 **[validate propagation setup]** |
| S/4 | `ZUI_CPC_PREF` RAP service | System-of-record API: reads via CDS, writes via actions (`saveChanges`, `endPreference`, `copyPreferences`, `applyAll`, `resolveComm`) |
| S/4 | BRFplus `ZCPC` | `FN_VALIDATE_CHANGE`, `FN_RESOLVE_COMM`, `FN_AUTO_PROCESS_EVENT` + decision tables (seed workbook, deliverable 3) |
| S/4 | Enhancements (sec. 6) | Move-in/out and BP-change handlers; outbound dispatch trigger |

**Service Cloud V2 configuration (tested)**

1. *Settings → Mashup Authoring → Create → URL mashup*
   URL: `https://<host>/cpc/` · parameters `mode=csr` (constant), `bp`, `ca`, `agent`, `caseId` (bound).
2. Case screen → *Adapt → Edit Master Layout* → add tab "Preferences" → embed the mashup →
   bind Account/BP → `bp`, logged-on user → `agent`, Case ID → `caseId` → publish for the CSR role.
3. Host requirements (production): serve from `https://svgbs.com/cpc/` (nginx config delivered)
   with `Content-Security-Policy: frame-ancestors <your Service Cloud host>` and **no**
   `X-Frame-Options`. Exact mashup parameter names **[validate in your tenant]**.

---

## 4. S/4 core (both options) — data + API

Delivered as compilable-style DDL in `s4-artifacts/01–03`. Summary:

| Artifact | Type | Notes |
|---|---|---|
| `ZCPC_PREFTYPE` | Table (customizing, delivery class C) | Master data: 24 types, scope BP/CA, allowed channels, defaults, mandatory flag — content = workbook sheet `DT_PREF_DEFAULTS` (single source: load table from workbook, mirror into BRF `DT_PREF_DEFAULTS` or read table from BRF via DB lookup — **decision: keep ONE source, recommend table + DB lookup**) |
| `ZCPC_PREF` | Table (application data) | Validity-dated records; `VKONT` initial for BP-scope; **half-open validity `[from, to)`** |
| `ZCPC_AUDIT` | Table (insert-only) | Full audit incl. `ON_BEHALF_OF`, `CASE_ID`, `CORRELATION_ID` |
| `ZI_CPC_PrefType` / `ZI_CPC_CustomerPref` / `ZI_CPC_Audit` | CDS interface views | `IsActive` computed with the half-open rule; association to `I_BusinessPartner` **[validate]** |
| `ZC_CPC_CustomerPref` | CDS consumption view | Transactional query, adds type texts |
| `ZUI_CPC_PREF` | Service definition + OData V4 binding | Actions: `saveChanges`, `endPreference`, `copyPreferences`, `applyAll`, `resolveComm` |
| DCL | Access control | CSR sees all; portal context restricted to own partner (subject→partner mapping table `ZCPC_USERMAP` — design decision) |

The `saveChanges` action implements: BRF validation → end-date → insert → audit →
raise event, in one LUW. Update/delete are **not** exposed anywhere (CPC03).

---

## 5. Integration Suite — iFlow inventory (Option A; iFlows 5–8 also in Option B)

| ID | iFlow | Direction | Sender → Receiver | Purpose / mapping | Error handling |
|---|---|---|---|---|---|
| IF_CPC_01 | `IF_CPC_Pref_Read` | Portal/SC2 → S/4 | HTTPS (from APIM) → OData V4 `Preferences` | Pass-through read with `$filter` on partner + (CA or scope BP); response trimmed to UI model | Sync; 503 + customer-safe body on S/4 down; 15 min response cache optional |
| IF_CPC_02 | `IF_CPC_Pref_SaveChanges` | Portal/SC2 → S/4 | HTTPS → OData action `saveChanges` | 1:1 payload (UI JSON already matches action schema); adds `correlationId` header; maps IAS subject → `changedBy` | Sync; maps RAP messages → per-change error array; **no retry** (user-driven; UI re-submits) |
| IF_CPC_03 | `IF_CPC_Pref_CopyApply` | SC2 → S/4 | HTTPS → actions `copyPreferences` / `applyAll` | CSR-only (scope check via JWT role) | Sync |
| IF_CPC_04 | `IF_CPC_ResolveComm` | Any consumer → S/4 | HTTPS → function `resolveComm` | Wraps BRF decision for dispatcher + previews | Sync; 300 ms target; circuit breaker → "suppress + alert" fallback (never send on failure) |
| IF_CPC_05 | `IF_CPC_Event_PrefChanged` | S/4 → downstream | Event Mesh `zcpc/PreferenceChanged` → OMS / DRMS / Marketing (per subscription) | Fan-out with per-target mapping; DNC changes duplicated to IF_CPC_06 | Async; exp. backoff ×5 → DLQ + Alert Notification |
| IF_CPC_06 | `IF_CPC_DNC_Broadcast` | S/4 → dialers | Event Mesh `zcpc/DncChanged` → outbound dialer / campaign systems | **Compliance-critical**: guaranteed delivery, sequence per partner | Async; aggressive retry; DLQ pages ops; delivery receipt logged to `ZCPC_AUDIT` via IF_CPC_02 pattern |
| IF_CPC_07 | `IF_CPC_Comm_Dispatch` | Trigger → providers | Event (BillCreated / OutageDetected / DREvent …) → `resolveComm` (IF_CPC_04) → Email/SMS/Voice/Print provider adapter | The "communicate per preference" flow; writes send/suppress + reason to audit | Async; per-channel retry; suppression is a *success* outcome (logged, not retried) |
| IF_CPC_08 | `IF_CPC_MasterData_Sync` | S/4 → BTP (opt.) | `ZCPC_PREFTYPE` delta → portal cache | Keeps portal type catalog current without hardcoding | Daily schedule + on-change event |

Common iFlow policies: OAuth2 client credentials (technical) or principal propagation
(user context) per destination **[validate]**; message logging with correlation ID;
payload PII minimization in logs (partner number yes, contact addresses no).

---

## 6. Enhancements — event-driven processing (both options)

| ID | Trigger | Mechanism | Handler logic |
|---|---|---|---|
| ENH-01 | **Move-Out completed** | IS-U event: Enterprise Event Enablement topic if available, else BOR event / BAdI in move-out processing **[validate per release — decide once in discovery]** | Call BRF `FN_AUTO_PROCESS_EVENT(MOVE_OUT)` → end CA-scope marketing/program/usage rows; billing retained until final bill; audit `SOURCE=BRF`; never touches BP-scope rows |
| ENH-02 | **Move-In completed** | Same mechanism as ENH-01 | `FN_AUTO_PROCESS_EVENT(MOVE_IN)` → seed CA defaults from `ZCPC_PREFTYPE` where no active record; BP-scope untouched |
| ENH-03 | **BP contact data changed** (phone/email — CPC05) | `BusinessPartner.Changed` event **[validate]** or BUPA BOR event | Re-validate channel addresses; flag records whose channel now lacks an address; notify customer per preference |
| ENH-04 | **Final bill settled** | FI-CA event / closing process hook **[validate]** | End remaining billing-category rows for the closed CA |
| ENH-05 | **RAP business event emit** | Event binding on `saveChanges` → Event Mesh **[validate RAP event → EM wiring per release]**; fallback: outbound call from behavior class via destination | Publishes `zcpc/PreferenceChanged` + `zcpc/DncChanged` |
| ENH-06 | **Outbound message trigger** | Billing/OMS/DRMS event consumed by IF_CPC_07 | Dispatcher must call `resolveComm` **at send time** (never cache decisions across DNC changes) |

---

## 7. Option B — S/4HANA CRM front end

```mermaid
flowchart LR
    subgraph S4[S/4HANA Utilities + Customer Management]
      IC[Interaction Center / WebClient UI\nZCPC assignment block on BP Overview]
      RAP[ZUI_CPC_PREF RAP service\n(same as Option A)]
      TAB[(ZCPC tables + CDS)]
      BRF[BRFplus ZCPC]
      ENH[Enhancements ENH-01..06]
    end
    IC -->|ABAP / OData local| RAP --> TAB
    RAP --> BRF
    ENH --> BRF
    RAP -. events .-> EM[Event Mesh] --> CPI[Integration Suite\nIF_CPC_05..08 only] --> OUT[Providers + downstream]
    PORT[Customer portal (still required for self-service)] --> CPI2[IF_CPC_01..02] --> RAP
```

**What changes vs Option A** — only the CSR presentation layer:

| Item | Option B specifics |
|---|---|
| UI | WebClient UI component `ZCPC` — assignment block on **BP Overview page** and IC agent inbox, per legacy specs: active-only default + **Show/Hide History** buttons (CPC01), **Copy** (CPC02), **End** (CPC03), **Apply All** (CPC04), BP phone/email maintenance from the block (CPC05), logged-on agent shown (CPC08) |
| Technology | BSP/WebClient component workbench (BOL/GenIL layer over `ZUI_CPC_PREF` or direct CDS consumption) — **[validate: S/4HANA for Customer Management add-on licensed & in your release; its roadmap position must be checked before committing]** |
| CSR auth | On-stack PFCG roles (`ZCPC_AUTH` object); no mashup/IAS needed for agents |
| Customer self-service | Still needs the portal front end (IF_CPC_01/02 path) — Option B does not remove that need |
| iFlows | IF_CPC_01–03 not needed for CSRs (in-stack UI); keep 01–02 for the portal; 05–08 unchanged |
| Effort delta | + WebClient UI development (largest single item, scarce skillset); − mashup/BTP UI hosting |

**GenIL/BOL sketch**: component `ZCPC` with root object `CustomerPref` mapped to
`ZI_CPC_CustomerPref`; actions delegated to the same RAP behavior (one write path for
both options — no logic duplication).

---

## 8. Configuration guide (step-by-step)

### Phase 0 — decisions & validation (1 week)
1. Confirm Option A or B (sec. 2). 2. Run every **[validate]** item against the landscape
(event mechanism per ENH-01/03/04 is the critical one). 3. Confirm master-data single
source (`ZCPC_PREFTYPE` + BRF DB lookup — recommended). 4. Sign off workbook rows
(deceased rule; RESEARCH in Move-Out set).

### Phase 1 — S/4 core (both options)
1. Package `ZCPC`; create tables from `01_zcpc_tables.txt` (+ indexes Z01/Z02).
2. Load `ZCPC_PREFTYPE` from workbook sheet `DT_PREF_DEFAULTS` (24 rows).
3. Create CDS views (`02_…`), DCL, consumption view + behavior + service (`03_…`);
   publish OData V4 binding; smoke-test `saveChanges` with the sample payload (sec. 4 of
   the BRFplus doc).
4. BRFplus: create application `ZCPC`, functions + decision tables; load
   `DT_DNC_POLICY` (11 rows) and `DT_EVENT_RULES` (7 rows) from the workbook;
   unit-test `FN_RESOLVE_COMM` for the three verified scenarios (opted-out; DNC voice
   fallback; DNC no-alternative suppress).
5. Enhancements ENH-01…04 per the validated mechanism; ENH-05 event emission.
6. PFCG: `ZCPC_AUTH` roles (CSR / admin / portal-technical); `ZCPC_USERMAP` for portal subjects.

### Phase 2A — BTP + Service Cloud V2 (Option A)
1. Subaccount: entitlements for Integration Suite, Event Mesh, Destination, Connectivity,
   (HTML5 repo or use svgbs.com host); Cloud Connector to S/4; IAS trust.
2. Destination `S4_CPC` (principal propagation **[validate]**; fallback OAuth2 technical
   user + `changedBy` from JWT claim).
3. Deploy iFlows IF_CPC_01…08; configure credentials, queues `zcpc/*` + DLQs, Alert
   Notification rules (DLQ non-empty, IF_CPC_06 failure = page).
4. Host CPC UI at production URL; set `API_BASE` to the APIM endpoint; CSP
   `frame-ancestors` = Service Cloud host (nginx config delivered).
5. Service Cloud V2: mashup + layout (sec. 3 steps — already rehearsed against the
   demo host).
6. End-to-end test script: sec. 10.

### Phase 2B — WebClient UI (Option B)
1. Component workbench: component `ZCPC`, BOL objects over the CDS views; assignment
   block on BP Overview; buttons per CPC01–CPC08.
2. IC profile + navigation bar entry; PFCG roles.
3. Portal path (IF_CPC_01/02) as in 2A steps 1–4 (no mashup).

### Phase 3 — cutover
Migrate legacy preferences (CPC15–17 pattern): OMS/DNC/DRMS extracts →
`saveChanges` with `SOURCE=MIGRATION` (audit intact); reconcile counts; parallel-run
outbound suppression checks for one billing cycle before switching dispatch.

---

## 9. Security, compliance, operations (both options)

- Principal propagation end-to-end; `CHANGED_BY` is always the human user; CSR writes
  carry `ON_BEHALF_OF` + `CASE_ID`.
- DNC: BP-level single record; `zcpc/DncChanged` SLA to all dialers (regulatory);
  suppression decisions audited with reason codes; dispatcher resolves at send time.
- Audit: insert-only; archiving object with retention per legal (proposal: 7 years —
  confirm with compliance).
- Monitoring: CPI MPL + Alert Notification; S/4 application log object `ZCPC`;
  correlation ID from UI → APIM → CPI → RAP → audit row.
- NFR targets (proposed, not guaranteed): save p95 < 2 s; resolveComm p95 < 300 ms;
  event→downstream p95 < 60 s; DNC broadcast < 15 min to all consumers.

## 10. Test strategy (acceptance excerpts)

| # | Case | Expected |
|---|---|---|
| T1 | CSR saves opt-out via SC2 mashup | Old row end-dated (`valid_to` = today), new row inserted, audit has CSR user + case, event published |
| T2 | Portal customer of another BP calls API with foreign partner | 403/404 — DCL blocks; nothing written |
| T3 | Opt-out attempt on `PAST_DUE` | Rejected by `FN_VALIDATE_CHANGE` (mandatory) |
| T4 | DNC active + voice message type | `resolveComm` returns fallback channel + reason `DNC_FALLBACK`; audit shows suppression path |
| T5 | DNC active + no permitted alternative | `send=false`, `DNC_NO_ALTERNATIVE`; nothing dispatched |
| T6 | Move-Out event | Marketing/program/usage CA rows ended; billing retained; BP rows untouched; `SOURCE=BRF` audit |
| T7 | Move-In on second CA | Defaults seeded for CA scope only; DNC remains single BP record |
| T8 | Record ended today | `IsActive = ''` today (half-open rule) — regression-guards the verified off-by-one |
| T9 | Event Mesh down during save | Save commits; event retried from outbox/queue; no user-facing failure |
| T10 | IF_CPC_06 delivery failure | DLQ + ops page; audit shows undelivered DNC broadcast |

## 11. Effort & plan (indicative — refine in Phase 0)

| Workstream | Option A | Option B |
|---|---|---|
| S/4 core (tables/CDS/RAP/BRF/enhancements) | 6–8 wks | 6–8 wks (same) |
| iFlows + Event Mesh | 3–4 wks | 2 wks (05–08 + portal path) |
| Front end | 1–2 wks (exists; productionize + mashup) | 5–7 wks (WebClient component) |
| Migration + test + cutover | 3–4 wks | 3–4 wks |
| **Indicative total** | **~13–18 wks** | **~16–21 wks** |

## 12. Open decisions

1. Option A vs B (sec. 2 recommendation: A).
2. Event mechanism per landscape for ENH-01/03/04 (single most important [validate]).
3. Master-data single source (recommended: `ZCPC_PREFTYPE` + BRF DB lookup).
4. Deceased-event policy row; RESEARCH in the Move-Out set (workbook flags).
5. Audit retention period; DNC broadcast SLA value.
6. Principal propagation vs technical user + claim (affects Destination + RAP auth design).
