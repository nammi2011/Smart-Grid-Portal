# SmartGrid Energy — Utility Customer Digital Portal

Live demos and complete solution documentation for a fictional utility's customer
digital experience: **Preference Center · Energy Program Enrollment · Rate Analyzer**,
designed for SAP BTP with integration to SAP S/4HANA Utilities.

## Live demos

| Demo | URL |
|---|---|
| **SmartGrid customer portal** — dashboard, preferences, program eligibility, rate analyzer, "what-if" simulator | https://nammi2011.github.io/Smart-Grid-Portal/ |
| **Customer Preference Center (CPC)** — Service Cloud V2 mashup-ready, 4 mock customers, BP-level DNC, BRF+ event simulation | https://nammi2011.github.io/Smart-Grid-Portal/cpc/ |

Both are single self-contained HTML files: no build, no dependencies, no network calls,
fictional data only. Open directly in a browser or serve from any static host.

Service Cloud V2 mashup URL template:

```
https://nammi2011.github.io/Smart-Grid-Portal/cpc/?mode=csr&bp={bp}&ca={ca}&agent={agent}&caseId={caseId}
```

## Repository layout

| Path | Content |
|---|---|
| `index.html` | SmartGrid portal demo |
| `docs/` | **Portal solution package** — all 29 deliverables (executive summary → BTP architecture → integration → data model → APIs incl. `openapi.yaml` → security/events → rate & eligibility engines → testing/CI-CD/roadmap), plus the combined Word edition |
| `cpc/` | **Preference Center**: live demo (`index.html`), full solution docs (`cpc/docs/` — BRFplus architecture, iFlow mapping specs, WebClient design, seed workbook), and S/4 artifacts (`cpc/s4-artifacts/` — DDL, CDS views, RAP service) |

Start with `docs/01-executive-summary-and-requirements.md` (portal) or
`cpc/README.md` (preference center). Combined Word editions:
`docs/SmartGrid-Portal-Complete-Package.docx` and
`cpc/docs/CPC-Complete-Package-Combined.docx`.

## Conventions

"SmartGrid Energy" is a fictional brand; all customers and data are mock. Rate math in
the demos is computed by a deterministic engine, never hard-coded. Standard SAP
artifacts named in the docs are marked **[validate]** against the target S/4HANA
landscape; no SAP APIs are invented.
