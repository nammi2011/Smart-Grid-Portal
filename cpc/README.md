# Customer Preference Center (CPC)

Embeddable preference-management UI + complete solution documentation.
Preferences maintained by customers (portal) or CSRs (SAP Service Cloud V2 mashup);
logic and events in SAP S/4HANA Utilities BRFplus; data in S/4 tables exposed via
CDS/RAP; Integration Suite iFlows for writes and event distribution.

## Live demo

**https://nammi2011.github.io/Smart-Grid-Portal/cpc/**

Service Cloud V2 mashup URL template:

```
https://nammi2011.github.io/Smart-Grid-Portal/cpc/?mode=csr&bp={bp}&ca={ca}&agent={agent}&caseId={caseId}
```

Mock customers (select by `bp`): `0000100045` Alex Johnson · `0001000234` Maria Garcia
(2 accounts, no DNC) · `0001000567` David Chen (DNC + DNM) · `0002000891` Riverside
Bakery LLC (commercial). `mode=customer` shows the self-service view. Demo data only —
saves preview the exact S/4 payload.

## Documentation (`docs/`)

| Document | Content |
|---|---|
| **CPC-Complete-Package-Combined.docx** | Everything below in one Word document — start here |
| cpc-solution-package (.md/.docx) | Options A (BTP + Service Cloud V2) & B (S/4 CRM), iFlow inventory, enhancements, configuration guide |
| cpc-brfplus-architecture (.md/.docx) | BRFplus application `ZCPC`, storage options, full API catalog |
| iflow-IF_CPC_02 / 06 / 07 mapping specs (.md/.docx) | Save path · DNC broadcast · communication dispatch |
| iflow-IF_CPC_07-VM11-template-annex (.md/.docx) | Message templates, variables, producer contracts |
| cpc-optionB-webclient-design (.md/.docx) | S/4 CRM WebClient component design (+ Fiori fallback) |
| CPC-BRFplus-DecisionTable-Content.xlsx | Seed content: 24 preference types, DNC policy, event rules |
| cpc-architecture-diagrams.html | Theme-aware SVG interaction diagrams |

## S/4 artifacts (`s4-artifacts/`)

Table DDL (`ZCPC_PREFTYPE/PREF/AUDIT`) · CDS interface views · consumption view + RAP
behavior + service definition.

## Conventions

Half-open validity `[valid_from, valid_to)` · BP-level DNC/DNM/third-party (apply to
all contract accounts) · end-date, never delete · one core class writes everything ·
outbound decisions resolved per message at send time. Standard SAP artifacts are marked
**[validate]** against the target landscape; no SAP APIs are invented.
