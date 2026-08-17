# Deliverable 1–2 — Executive Summary & Business Requirements

## 1. Executive summary

SmartGrid Energy customers today interact with fragmented systems: preferences live in
CRM, program enrollment in a demand-response management system (DRMS), rates in SAP
IS-U/S/4HANA Utilities, and usage in the MDMS. The **Utility Customer Digital Portal**
consolidates these into a single, consumer-grade digital experience — comparable to
leading banking and telecom self-service — delivered on **SAP Business Technology
Platform (BTP)** and integrated with **SAP S/4HANA Utilities**.

The portal delivers three capabilities on one customer-centered surface:

- **Customer Preference Center** — one place to manage communication channels,
  billing/paperless, usage alerts, outage notifications, program/marketing
  communications, and privacy consents, with a complete audit trail suitable for
  regulatory review (TCPA/CAN-SPAM/CCPA-style obligations).
- **Energy Program Enrollment Center** — a marketplace where customers discover
  efficiency, demand-response, EV, renewable, and assistance programs; check
  eligibility instantly against a configurable rules engine; and enroll with a
  tracked, auditable workflow.
- **Rate Analyzer & Recommendation Center** — computes each eligible tariff against
  the customer's actual 12–24-month usage (including AMI interval data), explains the
  cheapest option in plain language, lets the customer run "what-if" simulations
  (e.g., "What if I charge my EV after 11 PM?"), and submits rate change requests.

**Business outcomes targeted:**

| Outcome | Mechanism |
|---|---|
| Reduced call-center volume | Self-service preferences, enrollment, rate questions |
| Higher program participation | Personalized, eligibility-filtered marketplace |
| Improved customer satisfaction (JD Power-style digital scores) | Single modern portal, transparent rates |
| Regulatory compliance | Consent audit history, preference-change auditability |
| Peak-load reduction | Demand response + TOU rate migration driven by the analyzer |
| Revenue protection | Explainable, deterministic rate math builds trust in tariff changes |

**Approach:** an MVP built as a React frontend + SAP CAP services on BTP Cloud Foundry
with HANA Cloud, integrating to S/4HANA Utilities through SAP Integration Suite and
BTP Destinations, evolving in later phases to event-driven integration (Event Mesh),
full AMI analytics, and an AI Energy Advisor grounded in the deterministic rate engine.

## 2. Business requirements

### 2.1 Personas

| Persona | Description | Portal access |
|---|---|---|
| **Residential customer** | Homeowner, renter, EV owner, solar customer, smart-thermostat customer, assistance participant | Customer portal (role `CUSTOMER`) |
| **Commercial customer** | Small business, large C&I, property manager, multi-site | Customer portal + account selector across premises |
| **CSR** | Contact-center agent | Agent view: read customer data, act on behalf of customer with audit attribution (role `CSR`) |
| **Program administrator** | Defines programs, eligibility rules, reviews enrollments | Admin services (role `PROGRAM_ADMIN`) |
| **Rate administrator** | Maintains rate plans, eligibility, reviews rate change requests | Admin services (role `RATE_ADMIN`) |

### 2.2 Functional requirements (condensed)

**FR-PREF — Preference Center**
- FR-PREF-01 Manage channels: email, SMS, phone, push, postal mail; designate a preferred channel.
- FR-PREF-02 Billing preferences: paperless, bill-ready, payment confirmation/reminder, past-due, AutoPay notices.
- FR-PREF-03 Usage alerts with thresholds: daily/weekly kWh, monthly bill $, projected bill $, peak demand kW.
- FR-PREF-04 Outage preferences per event (detected, ERT, crew dispatched, restored, planned) per channel (SMS, email, push, voice).
- FR-PREF-05 Program communication opt-ins (efficiency, DR events, EV, solar, renewables, rebates, tips).
- FR-PREF-06 Marketing opt-ins (recommendations, partner offers, energy products, research).
- FR-PREF-07 Privacy & consent: consent records with effective/expiration dates, policy acceptance, third-party sharing, analytics consent, smart-meter data consent; full consent history visible to the customer.
- FR-PREF-08 Every change writes an audit record: customer, account, category, previous value, new value, effective date, source, timestamp, user.
- FR-PREF-09 Explicit **Save Preferences** action with success confirmation.

**FR-PROG — Program Enrollment**
- FR-PROG-01 Marketplace of programs across five categories (efficiency, demand response, EV, renewable, assistance) with card layout and filters (All, Eligible for Me, Recommended, Enrolled, per-category).
- FR-PROG-02 Program card shows name, category, description, savings, incentive, eligibility summary, enrollment period, capacity, status; actions Learn More / Check Eligibility / Enroll Now.
- FR-PROG-03 Configurable rules-based eligibility engine over customer/premise/meter/usage attributes; outcomes: Eligible, Potentially Eligible, Additional Information Required, Not Eligible — each with reasons.
- FR-PROG-04 Enrollment workflow: select → eligibility check → review details → accept T&C → provide required info → submit → backend processing → confirmation. Statuses: Draft, Submitted, Under Review, Additional Information Required, Approved, Enrolled, Rejected, Cancelled.
- FR-PROG-05 "My Programs" shows enrollment history and status; cancellation where allowed.

**FR-RATE — Rate Analyzer**
- FR-RATE-01 Retrieve 12/24-month (or custom range) usage incl. TOU splits and interval data; retrieve current + alternative eligible rates with all pricing components.
- FR-RATE-02 Compare current vs. alternatives: annual cost, monthly average, savings $, savings %, eligibility, risk indicator.
- FR-RATE-03 Interactive simulation: EV charging time, thermostat, peak/off-peak shifts, weekend usage, solar, battery, future consumption — with real-time recalculation and charts.
- FR-RATE-04 Recommendation engine (deterministic MVP): recommended rate, estimated savings, confidence score, plain-language explanation ("~62% of your usage is off-peak…"). Architecture supports later ML models.
- FR-RATE-05 Rate change request with review workflow and status tracking.

**FR-GEN — General**
- FR-GEN-01 Home dashboard: balance, next payment, current rate, usage + trend, estimated bill, savings opportunities, recommended programs/rates, alerts, quick actions (Pay Bill, View Usage, Manage Preferences, Explore Programs, Analyze My Rate, Report Outage).
- FR-GEN-02 Multi-account selector (commercial/multi-site).
- FR-GEN-03 Notification center; customer profile.
- FR-GEN-04 CSR mirror view with on-behalf-of audit attribution.
- FR-GEN-05 Responsive: desktop / tablet / mobile; WCAG 2.1 AA-oriented.

### 2.3 Out of scope (MVP)

Payment execution (link-out to existing payment provider), outage map, move-in/move-out,
native mobile apps (responsive web only), multi-language (architecture i18n-ready).

### 2.4 Success metrics (proposed)

- ≥ 40% of preference changes self-served within 6 months.
- ≥ 15% uplift in program enrollment conversion vs. legacy channels.
- ≥ 5% of eligible EV customers migrate to EV-TOU within 12 months.
- Portal NPS ≥ +20; dashboard load < 3 s (design target, see NFRs).
