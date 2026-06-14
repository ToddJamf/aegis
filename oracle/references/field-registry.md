# Oracle — Field Registry
# Aegis stack 2026.06.13
# Bundled (local). Verified select payloads per tool, Jamf tenant (2026-04-29 / 04-30 / 05-04).
# Capability modules pull selects from here. Payload-design discipline: shared/source-routing.md.
# Updated when schema drifts (P_5005 / P_5068). get_object_metadata is the self-healing recovery
# path when a field here goes stale — see shared/error-handling.md §A1.
#
# NOTE (charter follow-up): Decision 9 targets shrinking this to confirmed-dead patterns + tenant
# quirks only, with get_object_metadata as the live discovery path. Ported intact-but-cleaned for now
# to avoid breaking capability selects; the shrink is a post-QA Dev optimization.

---

## `ask_scorecard`

Bulk-safe up to 10 company_ids per call.

```python
ask_scorecard(company_ids=[<gsids>], user_query=<query>)
```

Read for the brief: `overall_score`, `overall_label`, `scorecard_name`, and **`overall_score_history`**
(drives the health trend arrow — see capability-brief.md). Strip: `scorecard_id`,
`scorecard_description`, `metadata`, `measures_flat` (often empty in tenant), `relationships`, `warnings`.

---

## `fetch_cta_list`

Always filter `IsClosed=false`. Always specify select. Supports linked objects (run_query can't).

```python
where = {"conditions": [
    {"fieldName": "CompanyId", "value": [<gsid>], "operator": "EQ", "alias": "A"},
    {"fieldName": "IsClosed",  "value": [False],  "operator": "EQ", "alias": "B"}
], "expression": "A AND B"}
select = ["Gsid", "Name", "DueDate", "PriorityId__gr.Name", "StatusId__gr.Name",
          "TypeId__gr.Name", "ReasonId__gr.Name", "OwnerId__gr.Name", "TotalTaskCount"]
# OverdueTaskCount removed — P_5005 dead field (Jamf tenant, 2026-05-11)
```

Strip on render: `Comments` (RICHTEXTAREA — strip rich text before render), `MinTaskDate`, `NextTaskDueDate`.
**IsClosed must be boolean `False`, not string** — string silently returns all CTAs.

---

## `fetch_success_plan_list`

Filter `IsClosed=false` for active plans.

```python
select = ["Gsid", "Name", "DueDate", "PercentComplete",
          "StatusId__gr.Name", "SuccessPlanTypeId__gr.Name", "OwnerId__gr.Name"]
```

>3 active SPs → flag "Abnormal SP volume — may include stale auto-created plans" (one account hit 7).

---

## `fetch_timeline_activity_list`

**Critical:** filter on `GsCompanyId` (NOT `CompanyId`). Prefer the semantic `contextual_user_query`
entry point (source-routing: timeline) over blunt date windows.

### Per-company timeline
```python
where = {"conditions": [
    {"fieldName": "GsCompanyId", "value": [<gsid>], "operator": "EQ", "alias": "A"}
], "expression": "A"}
select = ["Subject", "Gsid", "ActivityDate", "TypeName",
          "GsCreatedByUserId__gr.Name AS AuthorName", "NotesPlainText AS Notes"]
contextual_user_query = "<the meeting purpose>"   # semantic ranking + auto-RAG
limit = 10
```

### Author-filtered timeline (team theme extraction)
```python
where = {"conditions": [
    {"fieldName": "GsCreatedByUserId", "value": [<gsuser_gsids>], "operator": "IN", "alias": "A"},
    {"fieldName": "ActivityDate", "value": ["<60d_ago>T00:00:00Z"], "operator": "GTE", "alias": "B"}
], "expression": "A AND B"}
select = ["Subject", "ActivityDate", "TypeName", "Ant__Sentiment__c",
          "GsCompanyId__gr.Name AS CompanyName",
          "GsCreatedByUserId__gr.Name AS AuthorName", "NotesPlainText AS Notes"]
contextual_user_query = "<theme-extraction prompt>"
limit = 40
```

**SELECT pitfalls (P_5068):** don't use `ActivityTypeId__gr.Name` (doesn't exist — use `TypeName`,
STRING). Don't use `Sentiment` — use `Ant__Sentiment__c` (PICKLIST A/B/C/D/F).
Strip on render: `Notes` (HTML/signatures, truncate 200 chars). Drop: `ConversationId`, `ExternalId`,
`CrmId`, `SfTaskId`, `SfEventId`, `MediaUrl`, `Trackers`, `From`, `To`, `ExternalAttendees`,
`InternalAttendees`, `DurationInMins`.
**Sentiment hygiene:** in a 40-entry sample (one CSM's team, last 60d) only ~7.5% had sentiment set.
High-signal when present, rarely is. Treat as nice-to-have, not load-bearing.

---

## `resolve_user`

Avoid for common first names (truncates on fuzzy match) — use `run_query` on `gsuser` last-name
CONTAINS (error-handling §A4). When using: strip `identities[]` unless the user wants Slack/SFDC links.

## `resolve_customer`

Returns `companies` and `relationships`. Default to company unless the user says "relationship"
(error-handling §B4). Strip identities arrays. Exact-prefix matches first (e.g. `Acme` → `Acme, Inc.`,
not an unrelated fuzzy match). Attributes mode returns ARR/CSM/Stage/Renewal/Segment in one call
(source-routing: account attributes).

---

## `run_query` on `gsuser` — identity by email
```python
select = ["Name", "Gsid", "Email", "Employee_Type__gc",
          "CS_Territory_Team__gc", "CS_Territory_Region__gc",
          "Manager__gr.Name", "Manager", "IsActiveUser", "Out_of_Office__gc"]
where = [{"name": "Email", "operator": "EQ", "value": [<email>], "alias": "A"},
         {"name": "IsActiveUser", "operator": "EQ", "value": [True], "alias": "B"}]
limit = 1
```

## `run_query` on `gsuser` — name lookup (replaces resolve_user for common first names)
```python
select = ["Name", "Gsid", "Email", "Employee_Type__gc", "CS_Territory_Team__gc", "Manager__gr.Name"]
where = [{"name": "Name", "operator": "CONTAINS", "value": [<lastname>], "alias": "A"},
         {"name": "IsActiveUser", "operator": "EQ", "value": [True], "alias": "B"}]
limit = 10
```

## `run_query` on `gsuser` — direct reports (down-tree)
```python
select = ["Name", "Gsid", "Email", "Employee_Type__gc", "CS_Territory_Team__gc",
          "CS_Territory_Segment__gc", "Manager__gr.Name", "Out_of_Office__gc"]
where = [{"name": "Manager", "operator": "EQ", "value": [<manager_gsid>], "alias": "A"},
         {"name": "IsActiveUser", "operator": "EQ", "value": [True], "alias": "B"}]
limit = 50
```
Multi-tier: walk one level at a time, cache intermediates.

---

## `run_query` on `company` — book overview (CSM scope)
```python
select = ["Name", "Gsid", "Csm__gr.Name AS CsmName", "Csm",
          "PreviousCsm__gr.Name AS PreviousCsmName",
          "Customer_Category__gc", "RenewalDate",
          "Renewal_Status__gc", "Arr", "ARR_Band__gc",
          "CSM_Sentiment__gc", "CSM_Sentiment_Last_Updated__gc",
          "Engaged_This_Period_by_CSM__gc", "Last_Timeline_Entry_Engagement__gc",
          "Churn_Risk_Identified__gc", "Number_of_Escalated_Renewals__gc"]
where = [{"name": "Csm", "operator": "EQ", "value": [<csm_gsid>], "alias": "A"}]
limit = 10
```
PICKLIST fields use direct selection (`_PicklistLabel` auto-returns) — don't append `__gr.Name`.
**Engagement proxies:** `Engaged_This_Period_by_CSM__gc` (compliance source of truth) and
`Last_Timeline_Entry_Engagement__gc` (last CSM-logged date). ARR always displays but must never
infer engagement cadence/tier.
**Dead field:** `Engagement_Model__gc` — P_5005 in Jamf tenant (confirmed UAT 2026-06-13). Do not use.

## `run_query` on `relationship` — product table (verified 2026-05-04)
```python
select = ["Name", "Gsid", "Cloud_Family__gc", "Product_Type__gc",
          "Paid_For_Quantity__gc", "Deployed__gc", "Total_Enrolled_Devices__gc",
          "Last_Login__gc", "Environment_Type__gc"]
where = [{"name": "CompanyId", "operator": "EQ", "value": [<gsid>], "alias": "A"}]
limit = 20
```
Product name = `Cloud_Family__gc_PicklistLabel` (fallback `Product_Type__gc`). Flag thresholds
(production only — exclude Sandbox/Demo/Trial): Deployed% <50% → ⚠️ · <10% or last login stale >60d → 🚨.
PICKLIST (`Cloud_Family__gc`, `Environment_Type__gc`, `Stage`, `Status`) → direct, no `__gr.Name`.
`TypeId` → LOOKUP, supports `__gr.Name`.

## `run_query` on `company` — aggregate rollups (FLL / compliance / stale / escalations)
Group-by + render-layer math. **Never `SUM(CASE WHEN ...)` — P_5026.** Skill READS compliance from the
Gainsight pre-computed boolean, never computes tier/region/period. Anchor every filter alias in
`where_filter_expression` when used — it overrides the implicit AND-join (error-handling §A5). Bulk
IN-clause safe size: ~10 GSIDs / 7 CSMs (error-handling §A6).

(Full per-view selects — compliance roll-up, stale list, escalation/heat-map, count-by-company —
retained from the verified set; same fields, same GSID constants below.)

---

## Picklist / status GSIDs (Jamf tenant, verified 2026-04-29 → 04-30)

Config/record identifiers — not PII. Skip `get_picklist_values`; use directly.

```python
PICKLISTS = {
  ("company","Customer_Category__gc"): {"Spotlight":"1I00HCSPROT534ECYXCDEDHUAI1UH78GBDLL",
     "Retain":"1I00HCSPROT534ECYXYM8IRPTAAB1THJQGS9","Grow":"1I00HCSPROT534ECYXR7F75RC1STNJQLGVQC",
     "React":"1I00J3DJ0LA31T2USJ89K64M3SRVSJMWHK70"},
  ("company","Renewal_Status__gc"): {"Open":"1I00VM2QFXVQWI47QSADYMUZ03VBM0WNWCE6",
     "Late":"1I00VM2QFXVQWI47QS4M5QGV3L25X4QPSZ7A","Escalated":"1I00VM2QFXVQWI47QSFAZ1SAZJY69GIITKHG",
     "Notice of Churn":"1I00VM2QFXVQWI47QSERM7FWPVP161VNXO15",
     "Late Escalated":"1I00S7LFYQ5DJNFTY2ZBXMOD710F5XYR7RWC","Churn":"1I00VM2QFXVQWI47QS1ZYZ7POCNFGT7KV5Y1"},
  ("company","Last_Engagement_Timeframe__gc"): {"<30 Days":"1I000AQ7CLXE6GC8C0MFNI0TJRD2VICP92ZZ",
     "30-60 Days":"1I000AQ7CLXE6GC8C0N2GZXLYT0NLL9NJBYY","60-90 Days":"1I000AQ7CLXE6GC8C0M6YL7WK5QWCLBJTNOW",
     ">90 Days":"1I000AQ7CLXE6GC8C0PO39X9H88V6CF59B8E"},
  ("company","Account_Status__gc"): {"Current Customer":"1I00D3F36I0KHC861HOKK23F3UVIZTX9YKL7"},
  ("company","CSM_Sentiment__gc"): {"A":"1I005ZWPK9ZBV2E3EMWRPESWDAWVK4R60KIZ","B":"1I005ZWPK9ZBV2E3EMWIB8BE7T1KHE98E7X1",
     "C":"1I005ZWPK9ZBV2E3EM0YVWMUSPYL0A8145Q8","D":"1I005ZWPK9ZBV2E3EMD6UR4UVM6OYR0SOZPH","F":"1I005ZWPK9ZBV2E3EMNUWCDC1JIBEZS20YAL"},
  ("company","ARR_Band__gc"): {"$10k or less":"1I00ONSM92NZS4XF7Q5CLYL0M3D6SU9AMDR9","$10k - $50k":"1I00ONSM92NZS4XF7QIRWZNOF3IFFF6YNTTV",
     "$50k - $100k":"1I00ONSM92NZS4XF7Q5DS0FHZUXUDW2KCYB7","$100k - $250k":"1I00ONSM92NZS4XF7QBPF2L1XWTVY15Y2H2Y",
     "$250k or more":"1I00ONSM92NZS4XF7Q8LN2ZJZM9AHDVBX6RR"},
}
```

**Escalation set (heat-map filter):** `[Escalated, Late Escalated]`. `Notice of Churn` = post-escalation
(different conversation). `Churn` = lost.

---

## Account team fields on `company` (verified 2026-06-13, get_object_metadata probe)

| Field | Type | Notes |
|-------|------|-------|
| `Csm__gr.Name AS CsmName` | LOOKUP | Primary CSM |
| `PreviousCsm__gr.Name` | LOOKUP | Transfer note when populated |
| `Account_Owner__gc__gr.Name AS AEName` | LOOKUP | AE / Account Executive |
| `Renewal_Rep__gc__gr.Name AS RenewalRepName` | LOOKUP | Renewal Rep |
| `Sales_Engineer__gc__gr.Name AS SEName` | LOOKUP | Sales Engineer — confirmed LOOKUP → gsuser |
| `Executive_Sponsor__gc__gr.Name AS ExecSponsorName` | LOOKUP | Executive Sponsor — confirmed LOOKUP → gsuser |
| `IA_Account_Executive__gc` | STRING | IA AE — plain text, not a gsuser LOOKUP |

**CSE (CS Engineer):** No dedicated person-field on `company`. CSE involvement tracked via `Open_CSE_Risk_Mitigation__gc` (BOOLEAN) and `CSE_Risk_Requests_Created__gc` (NUMBER) — identity only, not name. CSE name may live on the CTA object; not available as a direct account team member.

Add to brief select when full account team context is needed:
```python
"Account_Owner__gc__gr.Name AS AEName",
"Sales_Engineer__gc__gr.Name AS SEName",
"Executive_Sponsor__gc__gr.Name AS ExecSponsorName",
"Renewal_Rep__gc__gr.Name AS RenewalRepName",
"IA_Account_Executive__gc"   # STRING — display as-is
```

---

## Escalation / Risk fields on `company` (verified 2026-04-30)

| Field | Type | Notes |
|-------|------|-------|
| `Renewal_Status__gc` | PICKLIST | Primary escalation flag — GSIDs above |
| `Number_of_Escalated_Renewals__gc` | NUMBER | `>0` = boolean escalation |
| `Churn_Risk_Identified__gc` | BOOLEAN | CSM-flagged risk |
| `Churn_Risk_Level__gc` | NUMBER | 1–5, sparse |
| `Open_CSE_Risk_Mitigation__gc` | BOOLEAN | CSE actively mitigating |
| `SAI_Churn_Risk_Analysis__gc` | RICHTEXTAREA | Strip HTML |

## Renewal / financial fields (verified 2026-04-30)

| Field | Type | Notes |
|-------|------|-------|
| `RenewalDate` | DATE | No `__gc`. Days-to-renewal computed at render. |
| `Available_to_Renew__gc` | CURRENCY | ARR up for renewal (vs current `Arr`) |
| `Auto_Renewal__gc` | BOOLEAN | |
| `Open_Renewal_Opp_Count__gc` | NUMBER | |

## Engagement / Activity fields (verified 2026-04-30)

KPI compliance source — skill READS, never computes.

| Field | Type | Notes |
|-------|------|-------|
| `Engaged_This_Period_by_CSM__gc` | BOOLEAN | **Compliance source of truth** |
| `Last_Engagement_Timeframe__gc` | PICKLIST | GSIDs above |
| `Last_Timeline_Entry_Engagement__gc` | DATE | Last CSM-engagement entry |
| `CSM_Meetings_Last_90_Days__gc` | NUMBER | |

---

## Field type cheat-sheet (`company`)

| Pattern | Direct (no `__gr.Name`) | Use `__gr.Name` |
|---------|------------------------|-----------------|
| PICKLIST | `Customer_Category__gc`, `ARR_Band__gc`, `CSM_Sentiment__gc`, `Renewal_Status__gc`, `Tier__gc`, `CS_Segment__gc` | — |
| LOOKUP | — | `Csm`, `PreviousCsm`, `Account_Owner__gc`, `Renewal_Rep__gc` |

PICKLIST + `__gr.Name` = `P_5068` (error-handling §A4).

---

## Maintenance

New verified select → append with date. Schema drift → update + reference error-handling §A1
(get_object_metadata recovery). This is the field discipline of the skill.

---

*2026.06.13 — Ported from Sentinel v3.7 → Oracle. Scrubbed customer/employee example names and a named CSM-team sample (genericized to Acme / "one CSM's team") per repo confidentiality rule. Cross-refs fixed: efficiency.md (deleted) → source-routing.md; error-handling Section 3/9 → §A4/§A5; added §A1 metadata-recovery + §A6 bulk limits. Config/picklist GSIDs retained (records, not people). Added overall_score_history for the trend arrow. Shared between Oracle and Mentor.*
