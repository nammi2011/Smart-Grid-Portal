# VM-11 — Template & Variable Annex (IF_CPC_07 `Comm_Dispatch`)

**Version** 1.0 · **Status** For build · Annex to `iflow-IF_CPC_07-mapping-spec.md` §5

Defines, per message template: the variables, their **source fields in the producing
event payload**, formats, mandatory/fallback rules, and sample rendered content per
channel. This annex is the contract between the **event producers** (billing, FI-CA,
OMS, DRMS, usage monitor), the **iFlow mapping**, and the **template authors** in each
channel provider.

> **[validate]**: template IDs against the chosen providers; sample copy is draft —
> marketing/legal own final wording; SMS sender ID and legal footer per jurisdiction.

---

## 1. Conventions (apply to every template)

| Rule | Definition |
|---|---|
| Variable syntax | `{{var}}` in this annex; map to each provider's syntax at template creation |
| Currency | `amount` variables arrive as decimal string `"127.45"`, currency separate (`"USD"`); templates render `$127.45` — **formatting in the template, never in the iFlow** |
| Dates | Variables carry ISO `yyyy-MM-dd` (+ `HH:mm` local where noted); templates render locale format (`Aug 21` / `08/21/2026`) |
| Times | Always **recipient-local**, already converted by the producer (`…Local` suffix); templates never do timezone math |
| Account masking | `accountLast4` only — full contract account numbers never appear in any outbound message (PII rule) |
| Name | `firstName` from BP master data, resolved by S/4 alongside the address (resolveComm response extension `FIRST_NAME`); fallback `"Customer"` — never blank |
| Links | `portalLink` is a **deep link** built by the iFlow: `https://<portal-host>/<route>?ca=<hash>` — tokenized reference, never raw account number in the URL |
| Fallbacks | A **mandatory (M)** variable missing → message parked to DLQ `COMM_TEMPLATE_VAR_MISSING` (producer contract breach), never sent half-rendered. **Optional (O)** variables render their fallback |
| SMS budget | ≤ 320 chars rendered (2 GSM segments) incl. opt-out footer where required; samples below are counted |
| SMS footer | Marketing-class SMS append `Txt STOP to opt out` **[validate per jurisdiction]**; transactional/safety exempt |
| Voice | `TPL_*_VOICE` are IVR flow references; variables feed text-to-speech slots — same variable set as SMS |
| Locale | v1.0 en-US only; variable layer is locale-neutral so adding languages touches templates, not the iFlow |

### 1.1 Common variables (available to all templates)

| Variable | Source | M/O | Fallback |
|---|---|---|---|
| `firstName` | resolveComm `FIRST_NAME` | O | `Customer` |
| `accountLast4` | event `contractAccount` → last 4 digits (iFlow transform) | M (account-scoped msgs) | — |
| `serviceAddressShort` | event `premiseShortAddress` (producer-supplied, e.g. "123 Green St") | O | omit clause |
| `portalLink` | iFlow-built deep link (route per template below) | O | portal home |
| `utilityName` | static config `SmartGrid Energy` | M | — |
| `supportPhone` | static config | M | — |

---

## 2. Templates

### 2.1 `TPL_BILL_READY` — msgType `BILL_READY` (routine) · source `s4/billing/BillCreated`

| Variable | Source event field | Type/format | M/O | Fallback |
|---|---|---|---|---|
| `billAmount` | `data.totalAmount` | decimal string | M | — |
| `billCurrency` | `data.currency` | ISO 4217 | M | — |
| `dueDate` | `data.dueDate` | ISO date | M | — |
| `billPeriod` | `data.billingPeriodStart` + `End` | ISO dates → "Jul 2026" | M | — |
| `autoPayFlag` | `data.autoPayActive` | boolean | O | `false` |
| `portalLink` route | `/billing/{billRef}` (`data.billReference`, tokenized) | — | O | portal home |

- **Email subject:** `Your {{billPeriod}} bill is ready — {{billCurrency}}{{billAmount}}`
- **Email body (excerpt):** `Hi {{firstName}}, your bill for account ****{{accountLast4}} is ready. Amount due {{billCurrency}}{{billAmount}} by {{dueDate}}.{{#autoPayFlag}} AutoPay is scheduled — no action needed.{{/autoPayFlag}} View it: {{portalLink}}`
- **SMS (156 ch):** `SmartGrid: your {{billPeriod}} bill is ready. ${{billAmount}} due {{dueDate}} on acct ****{{accountLast4}}. View: {{portalLink}}`
- **Push title/body:** `Bill ready` / `${{billAmount}} due {{dueDate}}`

### 2.2 `TPL_PAY_CONFIRM` — `PAY_CONFIRM` (routine) · `s4/fica/PaymentReceived`

| Variable | Source | Format | M/O | Fallback |
|---|---|---|---|---|
| `payAmount` / `payCurrency` | `data.amount` / `data.currency` | decimal / ISO | M | — |
| `payDate` | `data.postingDate` | ISO date | M | — |
| `payMethod` | `data.paymentMethodText` (producer-localized: "bank transfer", "card ****1234") | string ≤ 30 | O | `your payment method` |
| `newBalance` | `data.accountBalance` | decimal | O | omit clause |

- **Email subject:** `Payment received — {{payCurrency}}{{payAmount}}`
- **SMS (149 ch):** `SmartGrid: we received your payment of ${{payAmount}} on {{payDate}} for acct ****{{accountLast4}}. Thank you.`

### 2.3 `TPL_PAY_REMIND` — `PAY_REMIND` (routine) · `s4/fica/PaymentDueReminder`

| Variable | Source | Format | M/O | Fallback |
|---|---|---|---|---|
| `dueAmount` / `dueCurrency` | `data.openAmount` / `data.currency` | decimal / ISO | M | — |
| `dueDate` | `data.dueDate` | ISO date | M | — |
| `daysUntilDue` | iFlow-computed: `dueDate − today` | int | O | omit |
| `portalLink` route | `/billing/pay` | — | O | portal home |

- **SMS (163 ch):** `SmartGrid reminder: ${{dueAmount}} is due {{dueDate}} on acct ****{{accountLast4}}. Pay online: {{portalLink}}. Already paid? Please disregard.`
- **Voice (TTS slots):** amount, due date, account last-4; flow `IVR_PAY_REMIND` **[validate]**

### 2.4 `TPL_PAST_DUE` — `PAST_DUE` (**regulatory**) · `s4/fica/PastDueNotice`

| Variable | Source | Format | M/O | Fallback |
|---|---|---|---|---|
| `pastDueAmount` / `currency` | `data.openAmount` / `data.currency` | decimal / ISO | M | — |
| `originalDueDate` | `data.dueDate` | ISO date | M | — |
| `noticeType` | `data.dunningLevelText` ("Reminder", "Final notice") | string | M | — |
| `actionDeadline` | `data.deadlineDate` | ISO date | M | — |
| `consequenceText` | `data.consequenceText` (producer-supplied, legal-approved per dunning level) | string | M | — |

- **Rule:** all variables **M** — a regulatory notice with a missing field is parked, never sent partial. Wording is **legal-owned**; templates locked (change control).
- **Email subject:** `{{noticeType}}: {{currency}}{{pastDueAmount}} past due on account ****{{accountLast4}}`
- **SMS (180 ch):** `SmartGrid {{noticeType}}: ${{pastDueAmount}} past due (was due {{originalDueDate}}) on acct ****{{accountLast4}}. Please pay or contact us by {{actionDeadline}}: {{supportPhone}}`

### 2.5 `TPL_OUT_DETECT` — `OUT_DETECT` (**safety**) · `oms/OutageDetected`

| Variable | Source | Format | M/O | Fallback |
|---|---|---|---|---|
| `outageArea` | `data.areaDescription` ("Springfield NE") | string ≤ 40 | O | `your area` |
| `detectedAtLocal` | `data.detectedAtLocal` | ISO date-time local | M | — |
| `etrLocal` | `data.estimatedRestorationLocal` | ISO date-time local | O | `We're assessing the situation.` clause |
| `outageId` | `data.outageId` | string | M | — |
| `safetyLine` | static: downed-line warning | fixed text | M | — |

- **SMS (218 ch):** `SmartGrid: power outage detected affecting {{serviceAddressShort}} at {{detectedAtLocal}}.{{#etrLocal}} Est. restoration {{etrLocal}}.{{/etrLocal}} Stay clear of downed lines. Updates: {{portalLink}} Ref {{outageId}}`
- Route `/outage/{outageId}`; safety class → no quiet-hours hold, provider priority lane.

### 2.6 `TPL_OUT_ETR` — `OUT_ETR` (safety) · `oms/RestorationEstimate`

Variables: `etrLocal` (M), `outageId` (M), `etrChangeReason` (O, fallback omit).
**SMS (136 ch):** `SmartGrid update: estimated restoration for your outage (ref {{outageId}}) is now {{etrLocal}}. Track: {{portalLink}}`

### 2.7 `TPL_OUT_RESTORE` — `OUT_RESTORE` (safety) · `oms/PowerRestored`

Variables: `restoredAtLocal` (M), `outageId` (M), `outageDurationText` (O — producer-formatted "3 h 20 min", fallback omit).
**SMS (150 ch):** `SmartGrid: power restored at {{serviceAddressShort}} as of {{restoredAtLocal}}. Still out? Check your breaker, then call {{supportPhone}}. Ref {{outageId}}`

### 2.8 `TPL_OUT_PLANNED` — `OUT_PLANNED` (routine) · `oms/PlannedOutage`

| Variable | Source | Format | M/O | Fallback |
|---|---|---|---|---|
| `plannedStartLocal` / `plannedEndLocal` | `data.windowStartLocal` / `EndLocal` | ISO date-time local | M | — |
| `workReason` | `data.workDescription` | string ≤ 60 | O | `scheduled maintenance` |
| `outageId` | `data.workOrderRef` | string | M | — |

**Email subject:** `Planned outage {{plannedStartLocal}} — {{serviceAddressShort}}`

### 2.9 `TPL_DR_EVENT` — `DR_EVENT` (time-critical) · `drms/EventDispatch`

| Variable | Source | Format | M/O | Fallback |
|---|---|---|---|---|
| `eventStartLocal` / `eventEndLocal` | `data.eventStartLocal` / `EndLocal` | ISO date-time local | M | — |
| `programName` | `data.programName` ("Peak Time Rewards") | string | M | — |
| `creditText` | `data.incentiveText` ("earn up to $2/kWh") | string ≤ 40 | O | omit |
| `optOutLink` route | `/programs/dr/{eventId}/opt-out` (`data.eventId`) | — | M | — |

- **SMS (215 ch):** `SmartGrid {{programName}}: conservation event {{eventStartLocal}}–{{eventEndLocal}}. Reduce usage to {{creditText}}. Your thermostat may adjust automatically. Opt out of this event: {{optOutLink}}`
- Per-event opt-out is a program **term** (thermostat DR spec) — `optOutLink` is mandatory.

### 2.10 `TPL_HIGH_USAGE` — `HIGH_USAGE` (routine) · `cpc/usage/HighUsage`

| Variable | Source | Format | M/O | Fallback |
|---|---|---|---|---|
| `periodText` | `data.periodText` ("yesterday", "this week") | string | M | — |
| `usageKwh` | `data.usageKwh` | decimal, template renders `1,234 kWh` | M | — |
| `thresholdKwh` | `data.thresholdKwh` (the customer's own threshold) | decimal | M | — |
| `comparisonText` | `data.comparisonText` ("42% above your typical Tuesday") | string ≤ 50 | O | omit |

**SMS (168 ch):** `SmartGrid alert: you used {{usageKwh}} kWh {{periodText}}, above your {{thresholdKwh}} kWh alert level.{{#comparisonText}} That's {{comparisonText}}.{{/comparisonText}} Tips: {{portalLink}}`

### 2.11 `TPL_BILL_PROJ` — `BILL_PROJ` (routine) · `cpc/usage/ProjectedBill`

| Variable | Source | Format | M/O | Fallback |
|---|---|---|---|---|
| `projectedAmount` / `currency` | `data.projectedAmount` / `data.currency` | decimal / ISO | M | — |
| `thresholdAmount` | `data.thresholdAmount` (customer's bill-alert threshold) | decimal | M | — |
| `billMonth` | `data.billingMonth` ("August") | string | M | — |
| `daysLeftInCycle` | `data.daysRemaining` | int | O | omit |

**SMS (176 ch):** `SmartGrid: your projected {{billMonth}} bill is ${{projectedAmount}}, above your ${{thresholdAmount}} alert level.{{#daysLeftInCycle}} {{daysLeftInCycle}} days left this cycle.{{/daysLeftInCycle}} Details: {{portalLink}}`

---

## 3. Producer payload contracts (summary)

The **M** variables above define each producer's minimum payload. Breach = DLQ
`COMM_TEMPLATE_VAR_MISSING` with the producer named in the alert — fix at the source,
then replay. Contracts to be countersigned by each producer team in Phase 0:

| Producer | Events | Owns fields |
|---|---|---|
| S/4 billing | BillCreated | amounts, dates, period, billReference, autoPay |
| FI-CA / dunning | PaymentReceived, PaymentDueReminder, PastDueNotice | amounts, due/deadline dates, dunning level + **legal-approved consequence text** |
| OMS | 4 outage events | area, times (already recipient-local), outageId |
| DRMS | EventDispatch | event window (local), program name, incentive, eventId |
| Usage monitor (CPC) | HighUsage, ProjectedBill | usage/projection numbers + the customer's own thresholds (from `ZCPC_PREF` alert settings) |

## 4. Change control

- Templates for `PAST_DUE` (regulatory): legal sign-off required per change; version
  recorded in the dispatch audit row (`NEW_VALUE` gains `:tplVersion`).
- All others: marketing owns copy; variable **schema** changes require a version bump
  here (VM-11 v1.x) and producer re-sign-off when a new M variable is added.
- Adding a language: new template variants per locale; variable layer unchanged (§1).
