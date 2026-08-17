# Deliverable 7–8, 17, 22–24 — SAP BTP Solution Architecture, Diagrams & Deployment

## 1. Logical architecture

```mermaid
flowchart TD
    C[Customer / CSR / Admin\nWeb & Mobile Browser] --> WZ[SAP Build Work Zone\n(optional launchpad)]
    C --> AR[Application Router\nauth, routing, CSRF]
    WZ --> AR
    AR --> FE[Frontend App\nReact + TypeScript (UI5 Web Components)]
    AR --> CAP[Backend Services\nSAP CAP Node.js]
    CAP --> HANA[(SAP HANA Cloud\nportal data: preferences, consents,\nenrollments, analyses, audit)]
    CAP --> EM[[SAP Event Mesh]]
    CAP --> DEST[Destination + Connectivity Services]
    DEST --> IS[SAP Integration Suite\nCloud Integration + API Management]
    IS --> S4[SAP S/4HANA Utilities / IS-U]
    IS --> MDMS[AMI Head-End / MDMS]
    IS --> DRMS[Program Mgmt / DRMS]
    IS --> OMS[Outage Management]
    IS --> MKT[Marketing / Engagement]
    IS --> PAY[Payment Gateway]
    EM --> IS
    IAS[SAP Cloud Identity Services\nIAS + AMS] -. OIDC .-> AR
    XSUAA[Authorization & Trust Mgmt\nXSUAA] -. JWT scopes .-> CAP
    BPA[SAP Build Process Automation\napprovals] --- CAP
    OBS[Application Logging + Alert Notification\n+ Cloud ALM] --- CAP
```

**Key decisions**

| Decision | Choice | Rationale |
|---|---|---|
| Runtime | Cloud Foundry | Mature CAP support, app router pattern, multitenant-ready |
| Backend | CAP Node.js | CDS data model = HANA schema + OData/REST services + built-in auth annotations |
| Frontend | React + TS + UI5 Web Components for React | Consumer-grade UX with Fiori/Horizon look; served via app router or Work Zone |
| Portal DB | HANA Cloud | Preferences/consents/enrollments are system-of-record **portal** data; S/4 remains system of record for BP/contract/billing |
| Integration | Integration Suite (Cloud Integration + API Management) | Mediation, mapping, throttling, API policies in front of S/4 & MDMS |
| Async | Event Mesh | Decouple portal transactions from downstream provisioning |

## 2. SAP BTP service evaluation

| Service | Why needed | Mandatory? | Phase |
|---|---|---|---|
| Cloud Foundry Runtime | Hosts app router, CAP services, frontend | **Mandatory** | MVP |
| SAP HANA Cloud | Portal persistence (preferences, consents, enrollments, analyses, audit, aggregated usage cache) | **Mandatory** | MVP |
| SAP CAP (framework) | Backend programming model | **Mandatory** | MVP |
| Authorization & Trust Mgmt (XSUAA) | JWT validation, scopes, role collections | **Mandatory** | MVP |
| SAP Cloud Identity Services (IAS/IPS) | Customer + workforce identity, OIDC, MFA; B2C-style customer identity | **Mandatory** | MVP |
| Destination Service | Managed connectivity metadata + credentials for S/4, MDMS, DRMS endpoints — never in code or browser | **Mandatory** | MVP |
| Connectivity Service (+ Cloud Connector) | Reach on-premise S/4HANA Utilities / OMS securely | **Mandatory when any target is on-premise** | MVP |
| SAP Integration Suite | Mediated integration, mapping, error handling, API management/policies | **Mandatory** for enterprise rollout (MVP may stub selected flows) | MVP→2 |
| SAP Event Mesh | Async events (preference changed, enrollment submitted, rate change) | Optional MVP, **recommended Phase 2** | 2 |
| SAP Build Work Zone | Central launchpad unifying portal + other SAP/custom apps | Optional | 2–3 |
| SAP Build Process Automation | Enrollment/rate-change approval workflows | Optional (MVP can use status-driven CAP logic) | 2 |
| Application Logging Service | Central structured logs w/ correlation IDs | **Mandatory** | MVP |
| Alert Notification Service | Ops alerts (integration failures, error-rate spikes) | Recommended | MVP |
| Cloud Transport Management | Controlled MTA transport DEV→TEST→QA→PROD | Recommended | MVP |
| SAP Cloud ALM | Ops monitoring, integration & exception monitoring | Recommended | 2 |

## 3. SAP BTP deployment architecture

```mermaid
flowchart LR
    subgraph SUB[BTP Subaccount per env: DEV / TEST / QA / PROD]
      subgraph CF[Cloud Foundry space]
        APR[approuter app\nxs-app.json routes]
        UI[static React build\n(html5 repo or approuter static)]
        SRV[portal-srv (CAP)]
        DBD[portal-db deployer (HDI)]
      end
      XS[(xsuaa instance)]
      DST[(destination instance)]
      CON[(connectivity instance)]
      HC[(HANA Cloud HDI container)]
      EMS[(event mesh instance)]
      LOG[(application-logs)]
    end
    APR --> UI
    APR --> SRV
    SRV --> HC
    SRV --> EMS
    SRV --> DST
    DBD --> HC
    XS -.bind.- APR & SRV
    DST -.bind.- SRV
    CON -.bind.- SRV
    LOG -.bind.- SRV & APR
```

- One **MTA** (`mta.yaml`, see `cap/`) builds approuter + srv + db-deployer + UI.
- **Multi-utility SaaS option**: CF multitenant application pattern (SaaS Provisioning
  service + tenant-aware CAP `@sap/cds-mtxs`) if the portal is offered to multiple
  utilities; single-tenant per-subaccount otherwise (assumed for MVP).

## 4. Environment & transport strategy

| Env | Subaccount | Purpose | Data |
|---|---|---|---|
| DEV | portal-dev | Feature development, mocked destinations allowed | Synthetic |
| TEST | portal-test | System integration tests vs. S/4 sandbox | Synthetic |
| QA | portal-qa | UAT, performance, security testing | Masked production-like |
| PROD | portal-prod | Live | Production |

Transport: `mbt build` → MTA archive → **Cloud Transport Management** route
DEV→TEST→QA→PROD with approval gate before PROD (see CI/CD doc).

## 5. Local development instructions (production CAP app)

```bash
cd cap
npm install
cds watch          # SQLite in-memory, mock auth, serves OData/REST on :4004
# frontend against CAP:
cd ../frontend && npm run dev   # Vite proxy /api → localhost:4004 (vite.config.ts)
```

`cds watch` uses `cds.requires.[development]` profiles — SQLite + mocked auth; the
same model deploys unchanged to HANA Cloud via HDI.

## 6. SAP BTP deployment instructions

```bash
# prerequisites: cf CLI v8, mbt, logged into the target subaccount/space
cd cap
npm install
mbt build                                   # produces mta_archives/portal_1.0.0.mtar
cf login -a <api-endpoint> -o <org> -s <space> --sso
cf deploy mta_archives/portal_1.0.0.mtar
# then in BTP cockpit:
# 1. Map role collections (SmartGrid_Customer/CSR/ProgramAdmin/RateAdmin/PortalAdmin)
#    to IAS user groups.
# 2. Create destinations: S4HANA_UTILITIES, MDMS_API, DRMS_API (OAuth2ClientCredentials
#    or PrincipalPropagation via Cloud Connector for on-premise).
# 3. Configure Event Mesh queues/subscriptions (see events doc).
# 4. Smoke test: approuter URL → login via IAS → dashboard loads.
```

Destination configuration is the **only** place backend credentials live; the browser
receives only same-origin approuter routes (`/api/**` forwarded with the user JWT).
Follow SAP's security recommendations for approuter route and Destination service
configuration (no wildcard destinations to arbitrary hosts, `forwardAuthToken` only to
trusted backends).
