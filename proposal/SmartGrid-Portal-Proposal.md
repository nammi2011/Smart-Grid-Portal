---
title: "Utility Customer Digital Portal"
subtitle: "Solution Proposal — Preference Center, Energy Program Enrollment & Rate Analyzer on SAP BTP"
author: "Prepared by: [Your name / company]  ·  Prepared for: [Client / utility name]"
date: "July 2026"
---

\newpage

# 1. Executive summary

Utility customers today are asked to navigate several disconnected systems: communication
preferences live in CRM, program enrollment in a demand-response management system, rates
and billing in SAP S/4HANA Utilities, and meter usage in the MDMS. The result is a
fragmented experience, avoidable call-center volume, and missed opportunities to move
customers onto better rates and beneficial programs.

We propose a single **Utility Customer Digital Portal** — a modern, consumer-grade
self-service experience delivered on **SAP Business Technology Platform (SAP BTP)** and
integrated with **SAP S/4HANA Utilities**. It unifies three high-value capabilities on one
customer-centered surface:

1. **Customer Preference Center** — one place to manage communication channels, paperless
   billing, usage/outage alerts, program and marketing opt-ins, and privacy consents, with
   a complete regulatory-grade audit trail.
2. **Energy Program Enrollment Center** — a marketplace where customers discover
   efficiency, demand-response, EV, renewable, and assistance programs, check eligibility
   instantly against a configurable rules engine, and enroll through a tracked workflow.
3. **Rate Analyzer & Recommendation Center** — computes each eligible tariff against the
   customer's actual 12–24 months of usage, explains the best-fit rate in plain language,
   and lets customers run "what-if" simulations such as *"What if I charge my EV after
   11 PM?"* before requesting a rate change.

The engagement is de-risked by design: a **working demonstration application** is available
today (live link below), and the production solution is delivered as an **SAP BTP blueprint**
(SAP CAP services on SAP HANA Cloud, integrated through SAP Integration Suite) that a
delivery team can evolve directly from the demo. Every financial figure the portal shows is
computed by a deterministic, unit-tested rate engine — never hard-coded — so customers and
regulators can trust the numbers.

**Headline outcomes targeted:** reduced call-center volume, higher program participation,
measurable peak-load reduction through TOU-rate and demand-response adoption, improved
digital satisfaction scores, and demonstrable regulatory compliance for consent and
preference management.

\newpage

# 2. The business challenge

| Problem today | Impact |
|---|---|
| Preferences, programs, rates, and usage live in separate systems | Confusing customer experience; low self-service adoption |
| No transparent way for customers to understand or compare their rate | Distrust of tariff changes; under-adoption of beneficial rates (TOU, EV) |
| Program discovery and eligibility are manual and opaque | Low enrollment in efficiency, demand-response, and assistance programs |
| Consent and preference changes are hard to audit | Regulatory exposure (consent, marketing, privacy obligations) |
| Peak demand is expensive to serve | Higher wholesale cost; grid stress during peak windows |

A single digital front door directly addresses each of these.

# 3. Proposed solution

## 3.1 Customer Preference Center

A complete preference-management experience across seven areas — communication channels
(with a preferred channel), billing and paperless, usage-threshold alerts (e.g. *"notify me
when my projected monthly bill exceeds $200"*), outage notifications per event and channel,
program and marketing opt-ins, and privacy/consent management with an immutable consent
history. Every change records the customer, account, category, previous and new value,
effective date, source, timestamp, and acting user — the audit trail regulators expect.

## 3.2 Energy Program Enrollment Center

A marketplace-style catalog across five categories — Energy Efficiency, Demand Response,
Electric Vehicle, Renewable Energy, and Financial Assistance. Each program card shows the
incentive, estimated savings, enrollment window, and capacity, plus an instant, rule-by-rule
eligibility result. A guided workflow takes the customer from eligibility check through terms
acceptance and information capture to submission, with status tracked end-to-end (Submitted →
Under Review → Enrolled) in a "My Programs" view. The eligibility engine is **configurable
data, not code**, so program administrators can adjust rules without a software release.

## 3.3 Rate Analyzer & Recommendation Center

The analyzer prices every eligible tariff — fixed, tiered, time-of-use, seasonal, and demand
— against the customer's real historical usage, then presents a side-by-side comparison with
estimated annual cost, monthly average, savings, eligibility, and a risk indicator. An
explainable recommendation names the best-fit rate with a confidence score and a plain-language
rationale. An interactive **"What if?" simulator** lets customers model EV charging windows,
thermostat setback, solar, battery, and usage growth, and watch every rate recalculate live.
The recommendation engine is deterministic for the MVP and architected to accept machine-learning
models in a later phase, always keeping the deterministic engine as the authoritative source
for dollar figures.

\newpage

# 4. Solution architecture (SAP BTP)

The target architecture follows current SAP reference patterns and keeps SAP S/4HANA
Utilities as the system of record for business partner, contract, installation, device, and
tariff master data. The portal owns only its transactional data (preferences, consents,
enrollments, analyses, requests, and audit).

| Layer | Components |
|---|---|
| Experience | Web & mobile browser · React with SAP UI5 Web Components · optional SAP Build Work Zone |
| Access | Application Router · SAP Cloud Identity Services (IAS) + Authorization & Trust Management (XSUAA) · OAuth 2.0 / OIDC |
| Services | SAP Cloud Application Programming Model (CAP, Node.js) · deterministic rate & eligibility engine · audit + events |
| Data & async | SAP HANA Cloud · SAP Event Mesh · Destination & Connectivity services |
| Enterprise integration | SAP Integration Suite → SAP S/4HANA Utilities, AMI/MDMS, DRMS, OMS, Marketing, Payment |

**Security by design.** SAP backend credentials are never exposed to the browser — the
Destination service holds all technical credentials, and the application router forwards only
the authenticated user's token to CAP services. Authorization is role-based *and*
instance-based: a customer can only ever access their own accounts, usage, preferences,
enrollments, and analyses. Data is encrypted in transit and at rest; consent history is
immutable; every sensitive write is audited.

**Integration is landscape-aware.** Exact SAP S/4HANA Utilities interfaces vary by release,
deployment model, and enabled scope. Each integration point is identified with its pattern
(synchronous API, asynchronous event, or batch replication); concrete SAP APIs are validated
against the target landscape during the discovery phase rather than assumed. No SAP APIs are
invented.

# 5. Why this approach

- **De-risked delivery.** A working demo exists now; the production design is a direct
  evolution of it, not a separate rewrite. The same rate/eligibility engine runs in the demo
  and in the CAP services.
- **Trustworthy numbers.** All costs, savings, and recommendations are computed by a
  deterministic, unit-tested engine — nothing is hard-coded, so results are explainable and
  reproducible.
- **Standards-based.** Built on SAP's own BTP services and integration patterns, so it fits
  a utility's existing SAP investment and operating model.
- **Configurable, not brittle.** Program eligibility and rate rules are data that
  administrators maintain — reducing change cost over the life of the solution.
- **Compliance-ready.** Consent and preference auditing are built in, not bolted on.

\newpage

# 6. Live demonstration & what is included

A fully interactive demonstration is available in the browser — login, dashboard, preference
center, program marketplace with live eligibility, rate analyzer, and the "what-if" simulator
all function against realistic mock data.

> **Live demo:** [paste the shared demo link here]
>
> *(Sign in with any email/password, or use "Sign in as Alex Johnson".)*

The delivery package accompanying this proposal includes:

- **Complete demonstration application** — React + TypeScript front end and a Node.js back
  end sharing the same rate engine.
- **Production SAP BTP blueprint** — SAP CAP data model and services, security model,
  deployment descriptors (MTA), and seed data.
- **Full documentation set** — solution architecture, functional design, UX specification,
  data model, API specification (OpenAPI), security and event architecture, testing and
  CI/CD strategy, and an implementation roadmap.

# 7. Deliverables

| Area | Deliverables |
|---|---|
| Strategy & design | Executive summary, business requirements, functional design, customer journeys, UX specification, wireframes |
| Architecture | SAP BTP solution architecture, architecture diagrams, integration architecture, S/4HANA Utilities and AMI/MDMS integration designs |
| Technical design | Data model & ER diagram, API design & OpenAPI spec, security architecture, event architecture, rate-engine design, eligibility engine, AI roadmap |
| Build | React front-end demo, back-end demo, SAP CAP application structure, BTP deployment files |
| Delivery | Local & BTP deployment instructions, testing strategy, CI/CD strategy, implementation roadmap, risk register, production-readiness checklist |

# 8. Implementation roadmap

| Phase | Scope | Indicative duration |
|---|---|---|
| Phase 1 — MVP | Dashboard, Preference Center (with audit), Program Marketplace & enrollment, basic Rate Analyzer, selected SAP integrations, identity & logging | 3–4 months |
| Phase 2 | Full AMI/MDMS interval integration, advanced rate analysis, automated eligibility from S/4, notifications, event-driven integration, CSR view | +3–4 months |
| Phase 3 | AI Energy Advisor, ML rate & program recommendations, predictive usage & budget alerts, advanced targeting | +4–6 months |

*Durations are indicative and refined during discovery based on landscape and scope.*

\newpage

# 9. Commercial approach

We recommend a phased engagement that delivers value early and defers larger investment until
the MVP has proven adoption:

1. **Discovery & landscape validation** (fixed fee) — confirm SAP S/4HANA Utilities interfaces,
   identity model, rate/tariff catalog, and program rules; finalize the Phase 1 backlog.
2. **Phase 1 MVP delivery** (fixed-scope) — the capabilities above, deployed to your BTP
   environment.
3. **Phases 2–3** (time & materials or fixed-scope per release) — advanced integration and AI.

Detailed pricing is provided in a separate commercial schedule once scope is confirmed in
discovery. SAP BTP and SAP HANA Cloud runtime consumption is licensed by the utility directly
through its SAP agreement.

# 10. Assumptions, risks & dependencies

| Item | Type | Mitigation |
|---|---|---|
| Exact S/4HANA Utilities APIs vary by release/scope | Risk | Discovery workshop; Integration Suite isolates the portal from interface differences |
| MDMS vendor API capabilities and limits | Dependency | Aggregation-first strategy tolerates batch-only MDMS |
| Tariff complexity beyond standard components (riders, net-metering true-up) | Risk | Component-extensible rate model, validated against the real tariff book |
| Consent/regulatory requirements differ by jurisdiction | Risk | Versioned consent model; legal review gate before launch |
| Customer identity linked to BP/account | Dependency | Verified account-linking flow at registration |
| Recommendation accuracy and trust | Risk | Deterministic engine + explanations + confidence; estimates clearly labeled as non-binding |

# 11. Next steps

1. Review this proposal and the live demonstration with your stakeholders.
2. Schedule a **discovery & landscape-validation** workshop (recommended 1–2 weeks).
3. Confirm Phase 1 scope and commercial schedule.
4. Mobilize the delivery team and stand up the BTP development environment.

---

*Prepared by [Your name / company]. "SmartGrid Energy" is a fictional utility brand used for
the demonstration; no real utility branding is used. All performance figures are proposed
design targets, not guaranteed platform performance. Financial estimates in the portal are
illustrative and are not a guarantee of future bills.*
