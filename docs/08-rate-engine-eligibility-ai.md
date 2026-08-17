# Deliverable 16–18 — Rate Engine Design, Program Eligibility Engine, AI Future Architecture

## 1. Rate calculation engine

Implemented in `frontend/src/services/rateEngine.ts` (demo) — the same logic ports to
the CAP service (`cap/srv/`) for production, where it runs server-side. **The engine is
deterministic and authoritative**: every figure shown in the UI is computed from the
monthly TOU consumption profile; nothing is hard-coded.

### Supported components

| Component | Formula (per month m) |
|---|---|
| Fixed customer charge | `fixed` |
| Flat energy | `kWh_m × price` |
| Tiered energy | `Σ_tiers kWh_in_tier × tier_price` (tiers applied to total monthly kWh) |
| Time-of-use | `peak_m×p_peak + offpeak_m×p_off + superoff_m×p_super` |
| Seasonal | component carries `season ∈ {ALL, SUMMER, WINTER}`; applied only in matching months (summer = Jun–Sep) |
| Demand charge | `peakDemandKw_m × price_per_kW` |

`monthlyCost(rate, m) = fixed + energy(rate, m) + demand(rate, m)`
`annualCost(rate) = Σ_m monthlyCost` · `savings = annual(current) − annual(alt)` ·
`savings% = savings ÷ annual(current)`.

### Demo tariff set (mock, illustrative prices)

| Rate | Type | Components |
|---|---|---|
| R1 Residential Standard *(current)* | TIERED | $12.00/mo; ≤600 kWh @ $0.14; >600 kWh @ $0.18 |
| R-TOU Residential Time-of-Use | TOU | $13.00/mo; peak (4–9 pm) $0.26; off-peak $0.12; super off-peak (11 pm–6 am) $0.09 |
| RT-EV EV Time-of-Use | TOU | $15.00/mo; peak $0.30; off-peak $0.13; super off-peak $0.06 — requires EV |
| R-SEA Seasonal Flat | SEASONAL | $11.00/mo; Jun–Sep $0.19; Oct–May $0.13 |
| GS-D Small Commercial Demand | DEMAND | $25.00/mo; energy $0.095; demand $9.50/kW — not eligible for residential (demonstrates eligibility filtering) |

### Simulation model (What-if)

The simulator transforms the monthly TOU profile, then re-runs the same engine:

- **EV charging window**: the EV load (default 300 kWh/mo, adjustable) is currently
  distributed 40% peak / 40% off-peak / 20% super off-peak (evening charging).
  Selecting **Overnight** moves the entire EV load to super off-peak.
- **Thermostat setback**: shifts 8% of peak kWh → off-peak.
- **Peak shift %**: manual additional peak → off-peak shift.
- **Solar kW**: reduces off-peak (daytime) kWh by `kW × 115 kWh/kW/mo` (capped at
  available kWh) — simplified production model.
- **Battery**: shifts up to 150 kWh/mo peak → super off-peak (charge overnight,
  discharge at peak).
- **Consumption growth %**: scales all periods.

Worked example (avg month 1,037 kWh split 30/45/25 peak/off/super):
R1 ≈ $12 + 600×.14 + 437×.18 ≈ **$174.7**; after "charge EV after 11 PM",
RT-EV ≈ $15 + 191×.30 + 347×.13 + 499×.06 ≈ **$147.3** → ~$27/mo, ~19% savings —
computed live by the engine, and covered by unit tests (`frontend/src/services/rateEngine.test.ts`).

### Recommendation engine (MVP deterministic)

1. Compute annual cost for every **eligible** rate; rank ascending.
2. Recommended = cheapest with savings > 2% (else "stay on current rate").
3. **Confidence score** = f(months of history [12→0.75, 24→0.9], data completeness,
   margin of savings vs. runner-up, behavioral-assumption sensitivity: recommendations
   that depend on simulated behavior change are capped at 0.7).
4. **Explanation** assembled from computed facts: off-peak share, seasonal shape,
   EV/solar flags — e.g. *"We recommend the EV Time-of-Use Rate because ~62% of your
   usage occurs during off-peak hours and you own an EV. Estimated savings: $X/yr."*
5. Risk indicator: TOU/demand rates = MEDIUM/HIGH (bill varies with behavior),
   flat/tiered = LOW; shown beside every alternative.

Future: ML ranking model (gradient-boosted / clustering on AMI load shapes) trained in
the analytics stack, **still constrained to explainable outputs and validated against
the deterministic engine** (the engine remains the source of truth for dollar figures).

## 2. Program eligibility engine

Rules are **data, not code** (`ProgramEligibilityRule` rows; demo:
`frontend/src/services/eligibility.ts` + rules in `mock/data.ts`).

```
Rule := { attribute, operator (EQ|NE|GTE|LTE|IN), value, failureOutcome, failureReason }
```

Attributes available to rules: customerType, serviceTerritory, zipCode, premiseType,
rateCategory, meterType, isSmartMeter, annualKwh, avgMonthlyKwh, peakDemandKw, hasEV,
hasSolar, hasBattery, hasSmartThermostat, incomeQualified, enrolledPrograms.

**Evaluation:** run all rules; outcome = worst failure class among failed rules:
any `NOT_ELIGIBLE` → **Not Eligible**; else any `INFO_REQUIRED` → **Additional
Information Required**; else any `POTENTIAL` → **Potentially Eligible**; else
**Eligible**. Every rule returns its pass/fail + human-readable reason, shown to the
customer (and to Program Admins for rule debugging).

Example (as in the master prompt):

```
IF customerType = RESIDENTIAL AND isSmartMeter = true AND hasSmartThermostat = true
THEN eligible for Smart Thermostat Demand Response
```

Capacity guard: `enrolledCount < capacity` checked at enrollment time (409/422 when
full → waitlist status in Phase 2). Admin UI (Phase 2) maintains rules; MVP seeds them
via configuration.

## 3. AI future architecture (Phase 3)

```mermaid
flowchart TD
    C[Customer] --> P[Portal / AI Energy Advisor chat]
    P --> ORCH[Advisor Orchestrator\n(BTP, e.g. CAP + SAP AI Core / GenAI Hub)]
    ORCH --> CTX[Customer Context Service\nprofile · preferences · programs]
    ORCH --> USE[Usage Data Service\naggregates + AMI features]
    ORCH --> RE[Deterministic Rate Engine\n(authoritative $ figures)]
    ORCH --> EL[Eligibility Engine]
    ORCH --> LLM[LLM\nexplanation · conversation · summarization]
    LLM --> ANS[Explainable recommendation\nwith engine-computed numbers]
    ANS --> C
```

**Grounding rules (non-negotiable):**
- The **LLM never invents tariff math**. All dollar amounts, savings, and eligibility
  outcomes come from the deterministic engines via tool/function calls; the LLM
  formats, explains, converses, personalizes.
- Example: *"Which rate is cheapest if I charge my EV at night?"* → orchestrator runs
  `rates/simulate {evChargingWindow: OVERNIGHT}` → LLM narrates the engine's result.
- Guardrails: no financial advice beyond utility tariffs; PII minimization in prompts;
  responses logged for QA; human handoff to CSR on low confidence.

Capability roadmap: personalized rate & program recommendations → bill explanation /
high-bill analysis (decompose bill deltas: weather, usage, rate, fees) → predictive
usage (forecast + budget alerts) → conversational Energy Advisor (natural-language
rate analyzer) → proactive advisor nudges (only where marketing consent allows).
