# AI Scope Annex — Customer Preference Center

**Version** 1.0 · **Status** Roadmap annex to the CPC solution package

The CPC is unusually well-positioned for AI because of two assets the core design
already provides: a **structured, insert-only preference and dispatch history** (clean
training and grounding data) and a **deterministic decision core** — BRFplus plus the
single-writer class — that AI can sit *on top of* without ever being trusted with a
compliance decision.

**Governing principle (same as the rate engine): the LLM proposes, explains and
translates; BRFplus decides; the core writes.**

## 1. Where the LLM sits — architecture

```mermaid
flowchart LR
    subgraph PROPOSE[AI layer — proposes & explains]
      NL[Customer natural language\n"stop calling me, keep outage texts"]
      LLM[LLM\nintent extraction · summarization]
      COP[CSR copilot\naudit-trail summary · Q&A]
    end
    subgraph DECIDE[Deterministic core — decides & writes]
      CONF[Human confirmation\nplain-language rendering]
      SAVE[saveChanges API\nidentity guard]
      BRF[BRFplus ZCPC\nvalidate · resolve · DNC policy]
      CORE[ZCL_CPC_CORE\nend-date + insert + audit]
      DB[(ZCPC_PREF / AUDIT)]
    end
    NL --> LLM -->|proposed changes JSON| CONF --> SAVE --> BRF --> CORE --> DB
    DB -->|read-only| COP
    LLM -. never writes, never decides send/suppress .-x DB
    EVT[Business events] --> BRF -->|send/suppress| DISP[Dispatch]
    DISP -->|outcomes| DB
```

Two structural guarantees follow from this placement:

1. **Every AI-originated change passes the full trusted path** — human confirmation,
   token-based identity, BRFplus validation, all-or-nothing save, audit row. An
   AI-proposed change is indistinguishable in governance terms from a clicked one.
2. **The send/suppress decision path contains no AI at all.** "The model decided to
   call a DNC customer" is not a defensible sentence; therefore the model cannot.

## 2. Use cases

### Wave 1 — near-term (grounded in data & contracts that exist today)

| # | Use case | How it works | Value |
|---|---|---|---|
| A1 | **Natural-language preference management** (portal + IVR later) | LLM maps free text to a proposed `saveChanges` payload → customer confirms a plain rendering → normal path executes | Self-service for customers who won't navigate toggle screens; accessibility |
| A2 | **CSR copilot** (Service Cloud V2) | Summarize audit history ("mail consent withdrawn Jan after billing dispute; DNC since Nov"), answer "why isn't this customer getting letters?", draft the case note after a change | Shorter handle time; fewer preference-related escalations. Check **Joule extensibility** first **[validate]** — may be configuration, not build |

### Wave 2 — needs 6–12 months of accumulated dispatch/audit history

| # | Use case | How it works | Value |
|---|---|---|---|
| A3 | **Channel & send-time propensity** (classic ML) | Learn from IF_CPC_07 outcome logs which channel/time each customer engages with → surfaced as *recommended defaults*, decision stays in BRFplus | Higher engagement, fewer wasted contacts |
| A4 | **Opt-out / fatigue risk** | Predict when one more marketing touch tips a customer into `PARTNER → OUT` or full DNC → throttle proactively | Protects the contact channel as an asset |

### Wave 3 — governance-mature

| # | Use case | How it works |
|---|---|---|
| A5 | **Compliance & ops anomaly detection** | Patterns in the audit + reconciliation streams: CSR bulk-change anomalies, opt-out spikes per campaign, recurring DNC drift at one target |
| A6 | **Template intelligence** | Tone/reading-level variants and translations of VM-11 templates; the variable layer is untouched. `PAST_DUE` (legal-owned) excluded |
| A7 | **AI Energy Advisor** (portal side, already designed) | "Which rate is cheapest if I charge my EV at night?" — deterministic rate engine authoritative, LLM explains |

## 3. Red lines (non-negotiable, state them to every stakeholder)

1. AI **never** makes the send/suppress decision — BRFplus only.
2. AI **never** writes preferences — everything funnels through `saveChanges` with
   explicit confirmation; the audit trail stays complete and attributable.
3. Regulatory content (`PAST_DUE` wording) and financial figures are outside
   generative scope entirely.
4. PII minimization in prompts; no training on customer data without the consent the
   CPC itself manages (`THIRD_PARTY`/analytics consents gate the propensity models —
   the system enforces its own AI data governance).

## 4. Platform & data readiness

| Concern | Choice |
|---|---|
| GenAI runtime | SAP AI Core / **Generative AI Hub** on BTP — same governance envelope as the rest of the solution; Joule for the Service Cloud surface **[validate tenant availability]** |
| Classic ML | HANA Cloud PAL / AI Core pipelines over the audit + dispatch history |
| Grounding | `ZCPC_AUDIT`, `ZI_CPC_*` CDS views, VM-10/VM-11 catalogs — already structured |
| Data readiness | IF_CPC_07 outcome logging (SENT/SUPPRESSED + reason) is the feature store for A3/A4 — **it accumulates from day one of go-live**, so Wave 2 becomes possible ~6–12 months later without retrofitting |

## 5. Effort signals (indicative)

A2 copilot (if Joule-extensible): weeks, mostly prompt + grounding config · A1 NL
preferences: 4–6 weeks incl. confirmation UX · A3/A4 propensity: 6–8 weeks once data
exists · A5–A7: scoped per case. All additive — nothing in Waves 1–3 changes the core,
the APIs, or the compliance model.
