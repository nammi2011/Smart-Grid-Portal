# Option B — WebClient Component Design (S/4HANA CRM Front End)

**Version** 1.0 · **Status** For build (Option B only) · Companion to `cpc-solution-package.md` §7

Technical design for the on-stack CSR front end: a **WebClient UI component `ZCPC`**
rendered as an **assignment block on the BP Overview page** and inside the
**Interaction Center**, implementing the legacy CPC specification behaviors
(CPC01–CPC08, CPC16) against the *same* S/4 core (tables, CDS, BRFplus, core class)
used by Option A — one write path, two front ends.

> **Prerequisite [validate — gate before any build]:** SAP S/4HANA for Customer
> Management (the CRM add-on providing the WebClient UI framework, GenIL/BOL, IC) must
> be licensed, installed, and release-compatible in your landscape, and its roadmap
> position acceptable to the client. If this gate fails, Option B is not viable as
> specified and the fallback is a Fiori ("My Preferences" list-report/object-page over
> `ZC_CPC_CustomerPref`) — noted in §12.

---

## 1. Architecture position — one core, two front ends

```
WebClient UI component ZCPC (views, context nodes)
        │  BOL entities
GenIL component ZCPC (ZCL_ZCPC_GENIL)
        │  delegates — NO business logic in GenIL
ZCL_CPC_CORE  ◄────────── same class called by the RAP behavior (Option A)
  ├─ validate (BRFplus FN_VALIDATE_CHANGE)
  ├─ save: end-date + insert + audit (1 LUW, half-open validity)
  ├─ copy / apply-all / end
  ├─ resolve_comm (BRFplus FN_RESOLVE_COMM)
  └─ raise events (zcpc/PreferenceChanged, DncChanged)
        │
ZCPC_PREF / ZCPC_AUDIT / ZCPC_PREFTYPE  +  BRFplus ZCPC
```

**Design rule B0:** `ZCL_CPC_CORE` is the only object that writes `ZCPC_PREF`/`ZCPC_AUDIT`
or raises events. The RAP behavior class and the GenIL class are both thin adapters over
it. This guarantees identical validation, audit, eventing, and the half-open validity
convention regardless of which UI saved.

## 2. GenIL component model (`ZCPC`)

Registered in the component set of the BP framework (table `CRMC_GIL_COMP` /
maintenance view; add to the component set used by the account application **[validate
exact set name in your S/4 CM release]**).

| Object | Kind | Key | Attributes (structure) | Notes |
|---|---|---|---|---|
| `PrefRoot` | Access/root object | `PREF_ID` (GUID) | `ZCPC_S_PREF`: partner, vkont, scope, pref_type, type text, category, channel, value, valid_from, valid_to, is_active, source, changed_by, case ref, mandatory flag | Read via CDS `ZI_CPC_CustomerPref`; **no direct table access** |
| `PrefAudit` | Dependent object | `AUDIT_ID` | `ZCPC_S_AUDIT` (mirror of `ZI_CPC_Audit`) | Relation `PrefRootAuditRel` (1:N) |
| `PrefType` | Dependent/value object | `PREF_TYPE` | From `ZI_CPC_PrefType` | Drives dropdowns + mandatory/channel rules in UI |
| `PrefSearch` | Query object | — | `ZCPC_S_SEARCH`: partner (M), vkont (O), include_history (O), category (O) | Default: active only (CPC01) |
| `PrefAdvSearch` | Dynamic query | — | adds date range, source, type | For the history view filters |

**Methods in `ZCL_ZCPC_GENIL` (subclass of `CL_CRM_GENIL_ABSTR_COMPONENT` [validate
exact base class in release])**: `GET_OBJECTS`, `SEARCH_OBJECTS` → CDS reads;
`MODIFY_OBJECTS` → **rejected** (framework modify disabled); all changes go through
GenIL **actions** mapped 1:1 to core methods:

| GenIL action | Core call | UI trigger |
|---|---|---|
| `SAVE_CHANGES` | `ZCL_CPC_CORE=>SAVE_CHANGES` (same structure as RAP action; source = `CSR`, changed_by = `SY-UNAME`, on_behalf_of = partner) | Save button |
| `END_PREF` | `=>END_PREFERENCE` | End button per row (CPC03) |
| `COPY_PREFS` | `=>COPY_PREFERENCES` | Copy dialog (CPC02) |
| `APPLY_ALL` | `=>APPLY_ALL` | Apply All dialog (CPC04) |
| `RESOLVE_PREVIEW` | `=>RESOLVE_COMM` | Communication preview (§4.6) |

## 3. WebClient UI component `ZCPC` — structure

| Artifact | Name | Purpose |
|---|---|---|
| Component | `ZCPC` | Runtime repository, window `MainWindow` |
| Component controller | `ZCPC/ComponentController` | Holds partner/account context; receives BP from the embedding component |
| Custom controller | `ZCPC/CuCoPrefs` | Shared context: `PREFS` (collection), `SEARCH`, `TYPES`, `AUDIT` |
| View 1 | `ZCPC/PrefOV` | **Assignment block** (table view, the main UI) |
| View 2 | `ZCPC/PrefDetail` | Row edit form (channel dropdown, value, valid-from) |
| View 3 | `ZCPC/CopyPopup` | CPC02 dialog: target contract account + effective date |
| View 4 | `ZCPC/ApplyAllPopup` | CPC04 confirm with account list |
| View 5 | `ZCPC/AuditList` | Change history of the selected preference |
| View 6 | `ZCPC/CommPreview` | Resolve preview panel (DNC-aware) |
| Context nodes | `PREF` (BOL `PrefRoot`), `TYPE`, `AUDIT` | Value nodes for UI flags (edit mode, history toggle) |

### 3.1 `PrefOV` assignment block — columns & behaviors

Columns: Preference (type text + `BP-wide` icon for scope=BP) · Channel (dropdown,
editable inline) · Opt-in (checkbox) · Valid from · Source (icon+text: PORTAL/CSR/OMS/
DRMS/BRF/MIGRATION) · Actions (End).

| Behavior | Spec | Implementation |
|---|---|---|
| Active only by default | CPC01 | `PrefSearch.include_history = ''` on load |
| **Show History / Hide History** buttons | CPC01 | Toolbar toggle re-runs query with `include_history='X'`; ended rows render with strikethrough type text + gray (P_STYLE cell design), sorted below active |
| End preference | CPC03 | Row action → confirm popup → `END_PREF`; row moves to history on refresh; **no physical delete anywhere** |
| Copy | CPC02 | Toolbar → `CopyPopup` (target CA dropdown from FI-CA accounts of the partner **[validate read: FI-CA API vs `FKKVKP` select in core]**) → `COPY_PREFS`; BP-scope rows excluded with an info message |
| Apply All | CPC04 | Toolbar → `ApplyAllPopup` listing all CAs → `APPLY_ALL`; BP-scope excluded |
| Update BP phone/email | CPC05 | Link "Update contact data" navigates to the standard BP contact view via component usage **[validate target component in S/4 CM]**; on return, `RESOLVE_PREVIEW` refreshes (address changes affect channel resolution) |
| Agent credentials shown | CPC08 | Block header: `Maintained as: SY-UNAME (CSR)` — same attribution written to audit |
| Mandatory types | — | Opt-in checkbox disabled for `IsMandatory='X'` rows, tooltip "Regulatory — cannot be opted out" |
| DNC banner | CPC16 | If active BP-scope DNC: message area banner "Do Not Call active since &1 — applies to all &2 contract accounts"; voice options in channel dropdowns suppressed-with-warning |
| Save semantics | — | Edits buffer in the BOL transaction context; **Save** triggers one `SAVE_CHANGES` (all-or-nothing, matching D1); success message `ZCPC 001` "Preferences saved (&1 changes)" |

### 3.2 Eventing inside the component

`PrefOV` subscribes to controller events: `PREFS_SAVED` (refresh + message),
`HISTORY_TOGGLED`, `CONTEXT_CHANGED` (BP overview account switch → re-query). The
component publishes `ZCPC_SAVED` as a component event so embedding components (IC
inbox, BP overview) can refresh dependent blocks.

## 4. Embedding

### 4.1 BP Overview page (account application)

- Component usage `ZCPC_Usage` added to the BP overview component via an
  **enhancement set** (never modify the SAP component) **[validate host component name
  in your S/4 CM release — classic CRM: `BP_HEAD`]**.
- Context binding: host's BP (BuilHeader) → `ZCPC/ComponentController` partner; the
  block queries on first expand (lazy load) per overview-page performance rules.
- Added to the overview page structure via UI configuration (role config key `ZCPC`),
  default collapsed, direct-edit enabled.

### 4.2 Interaction Center

- Same component reused in the IC session: navigation bar entry `ZCPC-PREFS`
  (nav bar profile of the utility IC business role), plus the assignment block on the
  identified account's overview.
- Confirmed-account context supplies the partner; without confirmation the view shows
  "Confirm an account first".
- Optional (Phase 2): IC alert on DNC active via the IC event framework / intent-driven
  interaction **[validate rule modeler availability]**.

## 5. Authorization

| Element | Value |
|---|---|
| Auth object | `ZCPC_AUTH` (fields: `ACTVT` 02/03, `PARTNER` range, `PRCAT` category) |
| Display role | `Z_CPC_DISPLAY` — ACTVT 03; End/Copy/Apply/Save hidden (view config per config key) |
| Maintain role | `Z_CPC_MAINTAIN` — ACTVT 02 |
| Admin (BRF rules) | `Z_CPC_RULES` — BRF workbench access to `ZCPC` application only |
| Check location | `ZCL_CPC_CORE` (not the UI) — UI hides, core enforces |

## 6. Messages & error handling

Message class `ZCPC`: 001 saved · 010 "Preference &1 is regulatory and cannot be opted
out" (from BRF validate) · 011 "Channel &1 not allowed for &2" · 020 "Ended — history
retained" · 030 DNC banner text · 040 "Copied to account &1" · 041 "BP-level
preferences apply to all accounts and were not copied" · 050 technical error with
correlation ID (from core, logged to application log object `ZCPC`).

All BRF validation failures surface as row-anchored messages in the message area;
save aborts entirely (D1 all-or-nothing — same as Option A).

## 7. What is explicitly *not* built in Option B UI

- No preference logic in BSP/GenIL layers (B0) — validation, defaulting, DNC fallback
  all remain in BRFplus/core.
- No customer self-service — the portal path (Option A UI in `mode=customer` + IF_CPC_01/02)
  remains required and unchanged.
- No direct table maintenance (SM30) for `ZCPC_PREF` — core-only writes.

## 8. Configuration & transport checklist

1. Enhancement set assignment (client) → create component enhancement for host component.
2. GenIL: register `ZCPC` in component set; test in GenIL BOL browser (`GENIL_BOL_BROWSER`).
3. UI component `ZCPC` + views + runtime repository; UI config for config key `ZCPC`
   (column set, edit mode) per business role.
4. Overview page config: add block to account/IC roles; nav bar entry for IC.
5. PFCG roles §5; assign to CSR test users.
6. Message class, application log object, number ranges (shared with core build).
7. Transport: workbench (GenIL/UI/core) + customizing (config keys, nav bar, roles).

## 9. Test cases (UI-specific — core cases T1–T10 in the solution package still apply)

| # | Case | Expected |
|---|---|---|
| B1 | Open BP overview, expand block | Active prefs only, lazy-loaded; agent shown in header (CPC08) |
| B2 | Show History | Ended rows appear struck-through below active (CPC01) |
| B3 | End a preference → confirm | Row end-dated via core; audit row with `SY-UNAME`/CSR; history shows it (CPC03) |
| B4 | Copy to second CA | CA-scope rows copied; message 041 for BP-scope exclusion (CPC02) |
| B5 | Apply All | All CAs updated; audit `APPLY_ALL` (CPC04) |
| B6 | Opt-out click on PAST_DUE | Checkbox disabled; forced attempt via BOL rejected with message 010 |
| B7 | DNC active partner | Banner 030; voice channel options flagged |
| B8 | Update phone via CPC05 link, return | Comm preview refreshes with new number |
| B9 | Display-only role | All actions hidden; direct action call rejected by core auth check |
| B10 | Save with one invalid row among three | Nothing saved (D1); three row-anchored messages max, save re-enabled |
| B11 | IC session without confirmed account | Block shows confirm prompt, no query fired |

## 10. Effort refinement (replaces the 5–7 wk placeholder)

| Work item | Est. |
|---|---|
| GenIL component + BOL browser tests | 1.5 wk |
| UI component (6 views, config, popups) | 2.5 wk |
| BP overview + IC embedding, roles, config keys | 1 wk |
| Messages, auth, polish, unit tests B1–B11 | 1 wk |
| **Total (assumes core exists per Phase 1)** | **~6 wk** |

## 11. Open items

1. **Gate [validate]:** S/4HANA for Customer Management availability, release, roadmap sign-off.
2. **[validate]** host component/enhancement-set names, GenIL base class, component set name in your release.
3. **[validate]** BP contact-data navigation target (CPC05) in S/4 CM.
4. Decide IC alerting scope (Phase 1 banner only vs rule-modeler alert).
5. Confirm display-role column set with the business (hide Source column for display users?).

## 12. Fallback if the §11.1 gate fails

Fiori elements **List Report / Object Page** on `ZC_CPC_CustomerPref` (OData V4 already
published): table with active/history toggle via `IsActive` filter default, actions
(`saveChanges`, `endPreference`, `copyPreferences`, `applyAll`) as annotations-driven
buttons, DNC banner via header facet. Loses the BP-overview embedding intimacy; gains
zero WebClient dependency. Effort ≈ 3 wk. Same core, so switching is presentation-only.
