# Deliverable 25–29 — NFRs, Testing Strategy, CI/CD, Roadmap, Risks & Production Readiness

## 1. Non-functional requirements (proposed design targets — not guaranteed SAP platform performance)

| Area | Target |
|---|---|
| Availability | 99.5% portal availability (business hours 99.9%); graceful degradation when S/4/MDMS down (cached data + banner) |
| Performance | Dashboard initial load < 3 s; API p95 < 2 s (sync); rate analysis < 5 s on cached aggregates; simulator recalculation < 300 ms (client-side) |
| Scalability | 100k registered customers, 5k concurrent sessions, seasonal 3× bill-day peaks; CF horizontal scaling + HANA workload classes |
| Security | See security doc; OWASP ASVS L2; annual pen test |
| Accessibility | WCAG 2.1 AA-oriented; keyboard + screen-reader tested flows |
| Data privacy | Consent-gated marketing; PII masking in logs; retention per data model doc; DSR (access/erasure) support Phase 2 |
| Disaster recovery | RPO ≤ 4 h (HANA backups), RTO ≤ 8 h; MTA redeploy runbook; multi-AZ by BTP region |
| Observability | Structured logs w/ correlation IDs (Application Logging), metrics + alerts (Alert Notification), integration monitoring (Cloud ALM) |
| Auditability | 100% of preference/consent/enrollment/rate-change writes audited; immutable consent history |
| API reliability | Idempotent PUT/POST-with-key; retries w/ backoff; circuit breakers; DLQ for events |
| Mobile responsiveness | Full function 375px–1920px |

## 2. Test strategy

| Layer | Scope | Tooling (demo → prod) |
|---|---|---|
| Unit | **Rate engine** (each component type, TOU splits, seasonal boundaries, tier boundaries, simulation transforms), eligibility engine outcomes, reducers | Vitest (`frontend/src/services/rateEngine.test.ts`) → Jest/Vitest in CAP |
| API | Contract tests vs OpenAPI; authz (customer cannot read another's account → 404); validation errors; 422 not-eligible; 409 duplicate enrollment | Supertest / CAP test, Newman |
| UI | Preference save + audit toast; enrollment wizard; simulator recalculation; navigation | Playwright |
| Integration | iFlow mappings, event delivery, DLQ path, S/4 sandbox round-trips | Integration Suite test suites |
| Failure injection | S/4 down → cached banner; MDMS timeout → "usage unavailable" + retry; event broker down → local commit + deferred publish | Chaos scripts in TEST |
| Security | JWT tampering, IDOR attempts, CSRF, dependency scanning | ZAP, npm audit/Snyk |
| Accessibility | axe automated + manual keyboard/screen-reader pass on all 22 screens | axe-core CI |
| Performance | Bill-day load profile, rate-analysis soak | k6 |

**Sample rate-engine unit cases (implemented in the demo):**
- Given 500 kWh peak + 700 kWh off-peak on R-TOU ($13 fixed, 0.26/0.12) → exactly
  `13 + 500×0.26 + 700×0.12 = $227.00`.
- Tier boundary: 600 kWh on R1 = `12 + 600×0.14 = $96.00`; 601 kWh adds one kWh at 0.18.
- Seasonal: June vs. October same kWh → prices 0.19 vs 0.13.
- Simulation: overnight EV window moves all EV kWh to super off-peak (totals conserved).

## 3. CI/CD

```
Git (trunk-based, PR reviews)
 → CI: lint + typecheck + unit tests + API tests + npm audit + SAST
 → build: vite build + mbt build (MTA)
 → deploy DEV (auto) → smoke tests
 → deploy TEST (auto) → integration + Playwright + axe
 → QA via Cloud Transport Management (manual promote) → UAT + perf
 → PROD via CTMS with production approval gate
 → post-deploy smoke + rollback = redeploy previous MTA version (blue-green optional via CF)
```

Recommended tooling: GitHub Actions (or SAP CI/CD service) for build/test; **Cloud
Transport Management** for QA→PROD governance; version pinning + SBOM; secrets only in
BTP service bindings.

## 4. Implementation roadmap

| Phase | Scope | Duration (indicative) |
|---|---|---|
| **1 — MVP** | Dashboard, Preference Center (with audit), Program Marketplace + enrollment (portal-side workflow), basic Rate Analyzer on monthly aggregates, mock/selected SAP integrations (profile + balance read, nightly usage load), IAS/XSUAA security, logging | 3–4 months |
| **2** | Full AMI/MDMS interval integration, advanced rate analysis + simulator parity server-side, automated eligibility attributes from S/4, notification dispatch (email/SMS/push honoring preferences), Event Mesh rollout, Build Process Automation approvals, CSR view, Work Zone integration | +3–4 months |
| **3** | AI Energy Advisor (GenAI Hub), ML rate/program recommendations, predictive usage & budget alerts, advanced program targeting, DSR automation, multi-utility SaaS option | +4–6 months |

## 5. Risks, assumptions, dependencies

| # | Item | Type | Mitigation |
|---|---|---|---|
| 1 | Exact S/4HANA Utilities APIs vary by release/scope | Risk | Early landscape workshop; every named API flagged "to be validated"; Integration Suite isolates portal from API differences |
| 2 | MDMS vendor API capabilities/limits unknown | Risk/Dependency | Aggregation strategy tolerates batch-only MDMS; on-demand interval optional |
| 3 | Tariff complexity beyond modeled components (riders, net metering true-up) | Risk | Rate model is component-extensible; validate against real tariff book in Phase 1 |
| 4 | Consent/regulatory requirements differ by jurisdiction | Risk | Consent model versioned; legal review gate before launch |
| 5 | Customer identity ↔ BP/account linking quality | Dependency | Verified account-linking flow (account no. + verification) in registration |
| 6 | DRMS integration timeline | Dependency | Portal-side enrollment workflow works standalone; provisioning added when DRMS ready |
| 7 | Data volumes from AMI | Risk | No raw AMI in portal DB (see AMI doc) |
| 8 | Recommendation trust/accuracy | Risk | Deterministic engine + explanations + confidence; disclaimer that estimates aren't bill guarantees |

## 6. Production readiness checklist

- [ ] IAS tenants configured (customer + workforce), MFA policy set, account-linking verified
- [ ] Role collections mapped; IDOR test pass (customer cannot access foreign account)
- [ ] Destinations created per env; no credentials in code; rotation runbook
- [ ] HANA Cloud sizing + backup schedule + restore tested
- [ ] Integration Suite iFlows deployed; error handling + retry verified; API policies (quota, spike arrest) active
- [ ] Event Mesh queues + DLQs + Alert Notification rules configured
- [ ] Audit logging verified for all preference/consent/enrollment/rate-change writes
- [ ] NFR evidence: load test report, DR drill, accessibility audit
- [ ] Customer-safe error messages + correlation IDs verified end-to-end
- [ ] CTMS transport route + PROD approval gate active; rollback rehearsed
- [ ] Legal sign-off: privacy policy version, consent texts, tariff estimate disclaimers
- [ ] Runbooks: integration outage, DLQ drain, bill-day scaling, incident comms
