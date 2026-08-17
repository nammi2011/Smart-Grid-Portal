# BRFplus Step-by-Step Build Guide — Preference Center Application `ZCPC`

**Version** 1.0 · **Status** For build · Companion to `cpc-brfplus-architecture.md` (design)
and `CPC-BRFplus-DecisionTable-Content.xlsx` (seed rows). This document is the
**workbench-level configuration guide**: every application, data object, decision table,
expression, function, and ruleset — including the if/else logic written out rule by rule.

> Workbench labels vary slightly by S/4 release; where a label or capability is
> release-dependent it is marked **[validate]**. Build order matters — follow the steps
> in sequence, since later objects reference earlier ones.

---

## Step 0 — Prerequisites

| Item | Value |
|---|---|
| Transaction | `BRF+` (or `BRFPLUS`) — the BRFplus workbench |
| User mode | Set your workbench user mode to **Expert** (Workbench → Personalize) so all object types are visible |
| Authorization | BRFplus power-user role (FDT authorizations) **[validate your role assignment]**; developer key for the package |
| Package | `ZCPC` (same package as tables/CDS from `s4-artifacts/`) |
| Transport | Workbench request for the application + rules **structure**; decide per table whether **content** is transported (rule tables: yes) — customer preference *records* never live in BRFplus in Option A |
| Dependencies | Tables `ZCPC_PREF`, `ZCPC_PREFTYPE` exist (Option A DB lookups read them) |

**Naming conventions used throughout:** `EL_*` elements · `TS_*` structures · `TT_*`
table types · `DT_*` decision tables · `FE_*` formulas · `BO_*` boolean expressions ·
`LO_*` loops · `PC_*` procedure calls · `FN_*` functions · `RS_*` rulesets.

---

## Step 1 — Create the application

1. Workbench → **Create Application**.
2. Name: `ZCPC` · Text: *Customer Preference Center* .
3. **Storage type: Customizing** — rules are configuration, maintained per client and
   transported via customizing requests. (Do **not** choose Master Data storage in
   Option A; preference *records* live in `ZCPC_PREF`, not in BRFplus.)
4. Development package: `ZCPC` · Software component: your custom component.
5. Application language / texts: EN primary.
6. Save. Everything below is created **inside this application** (Contained objects),
   so it transports and versions as one unit.

---

## Step 2 — Data objects

Create under *Application ZCPC → Create → Data Object*. Bind to DDIC types wherever
possible (**Element → Binding Type: DDIC Element**) so BRFplus inherits length, domain
values, and value helps.

### 2.1 Elements

| Element | Type / DDIC binding | Used for |
|---|---|---|
| `EL_PARTNER` | `BU_PARTNER` (C10) | Business partner |
| `EL_VKONT` | `VKONT_KK` (C12) | Contract account (initial = BP scope) |
| `EL_PREF_TYPE` | `ZCPC_PREFTYPE-PREF_TYPE` (C20) | Preference type key |
| `EL_SCOPE` | Text C2, listed values `BP`, `CA` | Scope |
| `EL_CHANNEL` | Text C10, listed values `EMAIL SMS VOICE PUSH POST ALL` | Channel |
| `EL_PREF_VALUE` | Text C3, listed values `IN`, `OUT` | Opt state |
| `EL_MSG_TYPE` | Text C20 | Message type for resolution (same domain as `EL_PREF_TYPE`) |
| `EL_MSG_CLASS` | Text C14, values `SAFETY REGULATORY TIME_CRITICAL ROUTINE` | Message class |
| `EL_EVENT_TYPE` | Text C12, values `MOVE_IN MOVE_OUT BP_CHANGE DECEASED FINAL_BILL` | BO event |
| `EL_SEND` | Boolean | Resolution: send? |
| `EL_REASON` | Text C24 | `OK / OPTED_OUT / DNC_FALLBACK / DNC_NO_ALTERNATIVE / DNM_FALLBACK / NO_ADDRESS / EXPIRED` |
| `EL_ACTION` | Text C14, values `SET END RETAIN SEED_DEFAULT REVALIDATE NO_ACTION` | Event-rule action |
| `EL_MANDATORY` | Boolean | Mandatory type flag |
| `EL_VALID_FROM`, `EL_VALID_TO` | Date (DATS) | Validity |
| `EL_TODAY` | Date | Evaluation date (context, defaulted by caller) |
| `EL_SEQNO` | Number (integer) | DNC sequence |
| `EL_ALLOWED_CH` | Text C60 | Comma list of allowed channels |
| `EL_FALLBACK1`, `EL_FALLBACK2` | Text C10 | Fallback channels |
| `EL_MSGNO`, `EL_MSGTEXT` | Number / Text C120 | Validation messages |

### 2.2 Structures

| Structure | Components |
|---|---|
| `TS_PREF` | EL_PARTNER, EL_VKONT, EL_SCOPE, EL_PREF_TYPE, EL_CHANNEL, EL_PREF_VALUE, EL_VALID_FROM, EL_VALID_TO |
| `TS_CHANGE` | EL_PREF_TYPE, EL_SCOPE, EL_VKONT, EL_CHANNEL, EL_PREF_VALUE, EL_VALID_FROM (mirrors one `changes[]` line of the API) |
| `TS_TYPECFG` | EL_PREF_TYPE, EL_SCOPE, EL_ALLOWED_CH, EL_CHANNEL (default), EL_PREF_VALUE (default), EL_MANDATORY |
| `TS_RESOLVE` | EL_SEND, EL_CHANNEL, EL_REASON, EL_PREF_TYPE |
| `TS_ACTION` | EL_ACTION, EL_PREF_TYPE *(or category selector)*, EL_SCOPE |
| `TS_MESSAGE` | EL_MSGNO, EL_MSGTEXT, EL_PREF_TYPE |

### 2.3 Table types

`TT_PREF` (of `TS_PREF`) · `TT_CHANGES` (of `TS_CHANGE`) · `TT_ACTIONS` (of `TS_ACTION`)
· `TT_MESSAGES` (of `TS_MESSAGE`).

---

## Step 3 — Decision tables

Create → Expression → **Decision Table**. For each: define condition columns (left) and
result columns (right); load rows from the workbook sheets (manual entry, or XML
import via *Additional Actions → Table Data Exchange* **[validate availability]**).

### 3.1 `DT_PREF_DEFAULTS` — type master (24 rows, workbook sheet `DT_PREF_DEFAULTS`)

| Setting | Value |
|---|---|
| Condition columns | `EL_PREF_TYPE` |
| Result columns | `EL_SCOPE`, `EL_ALLOWED_CH`, `EL_CHANNEL` (default), `EL_PREF_VALUE` (default), `EL_MANDATORY` |
| Match | **Single match** (Return: first match) — types are unique |
| Row example | `DNC` → `BP`, `VOICE`, `VOICE`, `OUT`, `false` |
| Row example | `PAST_DUE` → `CA`, `EMAIL,SMS,VOICE,POST`, `EMAIL`, `IN`, `true` |

> **Design decision reminder:** `ZCPC_PREFTYPE` (table) is the recommended single
> source; in that case build `DT_PREF_DEFAULTS` as a *DB lookup* on `ZCPC_PREFTYPE`
> instead of a decision table, and keep this decision table only if the business
> insists on maintaining defaults in the BRF workbench. **Pick one — never both.**

### 3.2 `DT_DNC_POLICY` — suppression & fallback (11 rows, workbook sheet `DT_DNC_POLICY`)

| Setting | Value |
|---|---|
| Condition columns | `EL_REASON`-style condition column `COND` (values `DNC_ACTIVE`, `DNM_ACTIVE`) · `EL_MSG_TYPE` |
| Result columns | `EL_CHANNEL` (suppressed), `EL_FALLBACK1`, `EL_FALLBACK2`, `EL_REASON` (code) |
| Match | **Single match, first match wins** — order rows exactly as the workbook `SEQ` column; the `*` catch-alls are entered as an **empty (initial) condition cell** for MSG_TYPE, placed last |
| Row example | `DNC_ACTIVE` + `PAY_REMIND` → suppress `VOICE`, fb1 `SMS`, fb2 `EMAIL`, reason `DNC_FALLBACK` |

### 3.3 `DT_EVENT_RULES` — BO-event auto-processing (7 rows, workbook sheet `DT_EVENT_RULES`)

| Setting | Value |
|---|---|
| Condition columns | `EL_EVENT_TYPE` |
| Result columns | `TS_ACTION` components: `EL_ACTION`, selector (category or type list — model as `EL_PREF_TYPE` pattern column), `EL_SCOPE` |
| Match | **Multiple match** — an event returns *all* matching action rows (Move-Out returns END + RETAIN rows). Set *Return: all matches* so the function result is `TT_ACTIONS` |
| Row example | `MOVE_OUT` → `END`, `CATEGORY IN (MARKETING, PROGRAM, USAGE)`, `CA` |
| Invariant row | Every row's scope column = `CA` except the explicit `DECEASED` row — **BP-scope types are never selected by account events** (enforced again in the ruleset, Step 7.3) |

### 3.4 `DT_MSG_CLASS` — message type → class (11 rows, from the IF_CPC_07 spec VM-10)

Condition `EL_MSG_TYPE` → result `EL_MSG_CLASS`. Single match. (Used by dispatch when
called through `FN_RESOLVE_COMM` extensions; optional in v1 if the class stays in CPI.)

---

## Step 4 — Data retrieval: DB lookup + activity check

### 4.1 `LK_ACTIVE_PREF` — DB lookup on `ZCPC_PREF`

Create → Expression → **Database Lookup**.

1. Table: `ZCPC_PREF`.
2. Conditions: `PARTNER = EL_PARTNER` · `PREF_TYPE = EL_MSG_TYPE` · `VKONT = EL_VKONT`
   *(see 4.3 for the BP-scope variant)*.
3. Result: **move matching rows to `TT_PREF`** (multiple rows possible — history +
   active candidates).

> **Half-open validity cannot be fully expressed in DB-lookup conditions** (no
> "VALID_TO > EL_TODAY **or initial**" combination in some releases **[validate]**).
> Therefore the lookup fetches candidates and the *formula* below picks the active one.

### 4.2 `FE_IS_ACTIVE` — the half-open validity formula

Create → Expression → **Formula**, result type Boolean. Formula text (BRFplus formula
language — functions available via the formula editor’s catalog):

```
( IS_INITIAL( EL_VALID_TO ) ) OR ( EL_VALID_TO > EL_TODAY )
```

**Convention (critical, tested in the demo):** validity is `[VALID_FROM, VALID_TO)` — a
record ended today is **inactive today**. `>` not `>=`. Also require
`EL_VALID_FROM <= EL_TODAY` when picking the active record (see loop 4.4).

### 4.3 `LK_ACTIVE_PREF_BP` — BP-scope variant

Same as 4.1 but condition `VKONT` **is initial** (BP-level rows carry no contract
account). Used for `DNC`, `DNM`, `THIRD_PARTY` — one record governs all accounts.

### 4.4 `LO_PICK_ACTIVE` — loop expression

Create → Expression → **Loop** over `TT_PREF`:
*For each row:* IF `FE_IS_ACTIVE` AND `VALID_FROM <= EL_TODAY` THEN assign row to
context structure `TS_PREF` (the active record) and **exit loop**. If the loop ends
without a hit, `TS_PREF` stays initial → treated as "no active preference" (defaults
apply per `DT_PREF_DEFAULTS`).

---

## Step 5 — Boolean helpers (the small if/else building blocks)

| Expression | Type | Logic |
|---|---|---|
| `BO_OPTED_IN` | Boolean | `TS_PREF-PREF_VALUE = 'IN'` |
| `BO_DNC_ACTIVE` | Boolean | Result of resolve-sub-lookup: active BP-scope `DNC` record exists with value `IN` (reuses `LK_ACTIVE_PREF_BP` + `LO_PICK_ACTIVE` with `EL_MSG_TYPE = 'DNC'`) |
| `BO_DNM_ACTIVE` | Boolean | Same pattern for `DNM` |
| `BO_CHANNEL_ALLOWED` | Boolean | Formula: `CONTAINS_STRING( EL_ALLOWED_CH , EL_CHANNEL )` **[validate exact formula function name in your release; alternative: procedure call to a one-line ABAP helper]** |
| `BO_IS_MANDATORY` | Boolean | `EL_MANDATORY = true` (from `DT_PREF_DEFAULTS` / type config) |

---

## Step 6 — Functions (signatures)

Create → **Function**. Mode: **Functional** (single result) for validate/resolve;
Functional with table result for events. Assign the ruleset (Step 7) as the top
expression (*Functional Mode → Assigned Ruleset*).

| Function | Context (input) | Result | Ruleset |
|---|---|---|---|
| `FN_RESOLVE_COMM` | `EL_PARTNER`, `EL_VKONT`, `EL_MSG_TYPE`, `EL_TODAY` | `TS_RESOLVE` | `RS_COMM` |
| `FN_VALIDATE_CHANGE` | `EL_PARTNER`, `TT_CHANGES`, `EL_TODAY` | `TT_MESSAGES` | `RS_VALIDATE` |
| `FN_AUTO_PROCESS_EVENT` | `EL_EVENT_TYPE`, `EL_PARTNER`, `EL_VKONT` | `TT_ACTIONS` | `RS_EVENT` |

Tip: create the functions **after** data objects but you may create them before the
rulesets and assign later — the workbench allows forward wiring either way.

---

## Step 7 — Rulesets: the if/else logic written out

Create → **Ruleset**, assign to the function. Each numbered rule below is one ruleset
rule (condition + operations). "Exit" = operation *Exit processing* so later rules
don't fire.

### 7.1 `RS_COMM` (for `FN_RESOLVE_COMM`) — the send-time decision

Ruleset variables: `TS_PREF`, `TS_TYPECFG`, `EL_ALLOWED_CH`, `EL_FALLBACK1/2`.

```
Rule 10  (initialize)
  IF   true
  THEN TS_TYPECFG = DT_PREF_DEFAULTS( EL_MSG_TYPE )        " scope, allowed, default, mandatory
       IF TS_TYPECFG is initial:  TS_RESOLVE-SEND = false
                                  TS_RESOLVE-REASON = 'UNKNOWN_TYPE'   → Exit

Rule 20  (read the governing record — scope-aware)
  IF   TS_TYPECFG-SCOPE = 'BP'
  THEN TT_PREF = LK_ACTIVE_PREF_BP ; TS_PREF = LO_PICK_ACTIVE
  ELSE TT_PREF = LK_ACTIVE_PREF    ; TS_PREF = LO_PICK_ACTIVE

Rule 30  (no record → default applies)
  IF   TS_PREF is initial
  THEN TS_PREF-PREF_VALUE = TS_TYPECFG default value
       TS_PREF-CHANNEL    = TS_TYPECFG default channel

Rule 40  (opted out → suppress)   — mandatory types can never be OUT (RS_VALIDATE)
  IF   NOT BO_OPTED_IN
  THEN TS_RESOLVE-SEND = false ; TS_RESOLVE-REASON = 'OPTED_OUT' → Exit

Rule 50  (DNC: voice suppression + fallback)
  IF   TS_PREF-CHANNEL = 'VOICE'  AND  BO_DNC_ACTIVE
  THEN read DT_DNC_POLICY( 'DNC_ACTIVE', EL_MSG_TYPE ) → EL_FALLBACK1/2, EL_REASON
       IF  EL_FALLBACK1 not initial AND BO_CHANNEL_ALLOWED(EL_FALLBACK1)
                                    AND NOT (EL_FALLBACK1='POST' AND BO_DNM_ACTIVE)
       THEN TS_RESOLVE-SEND=true ; CHANNEL=EL_FALLBACK1 ; REASON='DNC_FALLBACK' → Exit
       ELSE IF EL_FALLBACK2 passes the same three checks
       THEN TS_RESOLVE-SEND=true ; CHANNEL=EL_FALLBACK2 ; REASON='DNC_FALLBACK' → Exit
       ELSE TS_RESOLVE-SEND=false ; REASON='DNC_NO_ALTERNATIVE' → Exit

Rule 60  (DNM: postal suppression + fallback)  — mirror of rule 50 with
         CHANNEL='POST', BO_DNM_ACTIVE, DT_DNC_POLICY('DNM_ACTIVE', …),
         fallback must not be VOICE when BO_DNC_ACTIVE.

Rule 70  (happy path)
  IF   true
  THEN TS_RESOLVE-SEND = true ; CHANNEL = TS_PREF-CHANNEL ; REASON = 'OK'
```

*(Address resolution — email/phone from BP master — is done by the RAP wrapper around
this function, not inside BRFplus, so rules stay side-effect-free.)*

### 7.2 `RS_VALIDATE` (for `FN_VALIDATE_CHANGE`) — save-time checks

Loop (`LO_EACH_CHANGE`) over `TT_CHANGES`; per line:

```
Rule 10  type exists:        DT_PREF_DEFAULTS(line-PREF_TYPE) initial
                             → append message 001 'Unknown preference type &1'
Rule 20  mandatory opt-out:  BO_IS_MANDATORY AND line-PREF_VALUE='OUT'
                             → append message 010 'Preference &1 is regulatory and cannot be opted out'
Rule 30  channel allowed:    NOT BO_CHANNEL_ALLOWED(line-CHANNEL)
                             → append message 011 'Channel &1 not allowed for &2'
Rule 40  scope consistency:  cfg-SCOPE='BP'  AND line-VKONT not initial
                             → append message 012 'BP-level preference must not carry a contract account'
Rule 41                       cfg-SCOPE='CA' AND line-VKONT initial
                             → append message 013 'Contract account required'
Rule 50  validity window:    line-VALID_FROM < EL_TODAY - 30  OR  > EL_TODAY + 365
                             → append message 014 'Effective date outside allowed window'
```

Result: `TT_MESSAGES` empty ⇒ valid. The RAP action treats **any** message as a hard
reject of the whole request (single-LUW rule D1 — the UI shows each message per line).

### 7.3 `RS_EVENT` (for `FN_AUTO_PROCESS_EVENT`)

```
Rule 10  TT_ACTIONS = DT_EVENT_RULES( EL_EVENT_TYPE )      " multiple match
Rule 20  (safety invariant — belt and braces)
  Loop TT_ACTIONS: IF action-SCOPE = 'BP' AND EL_EVENT_TYPE <> 'DECEASED'
                   THEN delete the row      " account events never touch BP scope
```

The **caller** (event handler `ZCL_CPC_EVENT_HANDLER` → core class) executes the
actions: `END` = end-date matching CA rows (`VALID_TO = event date`); `SEED_DEFAULT` =
insert defaults for CA-scope types without an active record; `RETAIN` = no-op marker;
`REVALIDATE` = re-check channel addresses. BRFplus decides; ABAP acts.

---

## Step 8 — Simulate before you activate

Workbench → function → **Start Simulation**. Run at minimum these (they mirror the
tested UI behavior and the acceptance tests):

| # | Function | Input | Expected |
|---|---|---|---|
| S1 | RESOLVE | PAY_REMIND, customer opted-in VOICE, DNC active | SEND=true, CHANNEL=SMS (or EMAIL), REASON=DNC_FALLBACK |
| S2 | RESOLVE | RESEARCH, opted-in VOICE, DNC active | SEND=false, DNC_NO_ALTERNATIVE (per policy row 50) |
| S3 | RESOLVE | PARTNER, opted out | SEND=false, OPTED_OUT |
| S4 | RESOLVE | record ended **today** | treated inactive → default applies (half-open check) |
| S5 | VALIDATE | opt-out of PAST_DUE | message 010 |
| S6 | VALIDATE | scope BP + VKONT filled | message 012 |
| S7 | EVENT | MOVE_OUT | END rows for marketing/program/usage CA only; no BP rows |
| S8 | EVENT | MOVE_IN | SEED_DEFAULT rows CA-scope only |

Save each as a **stored test case** in the workbench so regression runs survive
transports **[validate feature name per release]**.

---

## Step 9 — Activate, transport, call

1. **Activate** bottom-up: data objects → expressions → decision tables → rulesets →
   functions (or use *Activate with all dependent objects* on the application).
2. Transport: application + structure on the workbench request; rule-table **content**
   rows on a customizing request (workbench prompts per object).
3. ABAP caller (used by the RAP behavior class and the event handler):

```abap
DATA(lo_fct) = cl_fdt_factory=>if_fdt_factory~get_instance(
                  )->get_function( iv_id = lv_fn_resolve_guid ).
DATA(lo_ctx) = lo_fct->get_process_context( ).
lo_ctx->set_value( iv_name = 'EL_PARTNER'  ia_value = iv_partner ).
lo_ctx->set_value( iv_name = 'EL_MSG_TYPE' ia_value = iv_msg_type ).
lo_ctx->set_value( iv_name = 'EL_TODAY'    ia_value = sy-datum ).
cl_fdt_function_process=>process( EXPORTING io_function = lo_fct
                                            io_context  = lo_ctx
                                  IMPORTING eo_result   = DATA(lo_result) ).
lo_result->get_value( IMPORTING ea_value = ls_resolve ).   " TS_RESOLVE
```

Store function **GUIDs** (not names) in a small config table `ZCPC_BRF_CFG` so renames
never break the caller. **[validate exact FDT API signatures on your release]**
Performance: BRFplus generates ABAP classes on first call after activation; warm them
in the transport pipeline (call once in a post-import method) to avoid first-user lag.

## Step 10 — Load the seed content

1. Open `CPC-BRFplus-DecisionTable-Content.xlsx`.
2. Fix the two rows flagged for business sign-off (deceased policy; RESEARCH in
   Move-Out set).
3. Enter rows per sheet into the matching decision table (or Table Data Exchange XML
   import **[validate]**). Row order matters only for `DT_DNC_POLICY` (first-match).
4. Re-run the Step 8 simulations — all eight must pass before transport.

## Governance

- **Change control:** rule-content changes ride customizing transports with the same
  approvals as tariff config; `DT_DNC_POLICY` changes additionally need compliance
  sign-off (they alter regulatory behavior).
- **Versioning:** BRFplus versions every activation — the workbench timeline is your
  rule audit; do not delete old versions.
- **Authorizations:** restrict workbench maintenance of `ZCPC` to role `Z_CPC_RULES`
  (FDT authorization objects **[validate object names]**); display for support roles.

## Build checklist (print this)

- [ ] App `ZCPC` (Customizing storage) in package ZCPC
- [ ] 19 elements · 6 structures · 4 table types (Step 2)
- [ ] `DT_PREF_DEFAULTS` (or DB-lookup variant — decided, not both) · `DT_DNC_POLICY` (ordered!) · `DT_EVENT_RULES` (multi-match) · `DT_MSG_CLASS` (optional)
- [ ] `LK_ACTIVE_PREF`, `LK_ACTIVE_PREF_BP`, `FE_IS_ACTIVE` (**strict `>`**), `LO_PICK_ACTIVE`
- [ ] 5 boolean helpers (Step 5)
- [ ] `FN_RESOLVE_COMM` / `FN_VALIDATE_CHANGE` / `FN_AUTO_PROCESS_EVENT` + rulesets 7.1–7.3
- [ ] Simulations S1–S8 green, saved as test cases
- [ ] Activated, transported, GUIDs in `ZCPC_BRF_CFG`, caller smoke-tested
- [ ] Seed content loaded from workbook after business sign-off
