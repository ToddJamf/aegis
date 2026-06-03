# Radar — Capability: Renewal Triage
# Aegis v2.0 — June 2026
# https://raw.githubusercontent.com/ToddJamf/aegis/main/radar/capability-renewal.md
#
# Tab 1 of the Radar artifact. 180-day renewal window, R1–R5 scoring, swim lanes.
# Load scoring formulas from references/scoring.md.
# Load HTML artifact template from references/artifact.md.

---

## Trigger

Renewal tab only: "show my renewals", "what's renewing", "renewal triage".
Full Radar (both tabs): bare "show my radar" / "radar" invocation.

---

## Data query

Load `references/scoring.md` before computing any score.
Load `references/artifact.md` before rendering the HTML.

### Company query (single query — serves both renewal and expansion tabs)

```python
run_query(
    object_name="company",
    select=[
        "Name", "Gsid",
        "Csm__gr.Name AS CsmName",
        "Primary_Territory_Owner__gr.Name AS TerritoryOwnerName",
        "Segment_Parent__gc", "Customer_Category__gc",
        "RenewalDate", "Renewal_Status__gc",
        "Available_to_Renew__gc", "Auto_Renewal__gc",
        "Open_Renewal_Opp_Count__gc",
        "Overall_Score__gc", "Churn_Risk_Identified__gc",
        "Churn_Risk_Level__gc", "Open_CSE_Risk_Mitigation__gc",
        "Last_Timeline_Entry_All__gc",
        "Last_Timeline_Entry_Engagement__gc",
        "Last_Timeline_Entry_EBR__gc",
        "Last_Timeline_Entry_Cadence_Call__gc",
        "Last_Timeline_Entry_CSM_Email__gc",
        "Expansion_Rediness_Level__gc",     # note: Gainsight typo — "Rediness" not "Readiness"
        "Staircase_Overall_Score__gc",
        "Staircase_Engagement_Score__gc",
        "Staircase_Sentiment_Score__gc",
        "Arr", "ARR_Band__gc",
    ],
    where=[
        {"name": "Account_Status__gc", "operator": "EQ",
         "value": ["1I00D3F36I0KHC861HOKK23F3UVIZTX9YKL7"], "alias": "A"},
        {"name": "True_Customer__gc", "operator": "EQ", "value": [True], "alias": "B"},
        {"name": <book_field>, "operator": "EQ", "value": [<gsid>], "alias": "C"},
        {"name": "Arr", "operator": "GTE", "value": [10000], "alias": "D"},
    ],
    where_filter_expression="A AND B AND C AND D",
    limit=1000
)
```

**No date filter at query time.** Tab 1 splits by `RenewalDate` in the render layer. Tab 2 uses all accounts. One query, two views.

### Relationship query (batch, fires after company query returns GSIDs)

```python
run_query(
    object_name="relationship",
    select=[
        "CompanyId", "Cloud_Family__gc", "Product_Type__gc",
        "Paid_For_Quantity__gc", "Deployed__gc",
        "Total_Enrolled_Devices__gc",
        "Total_Active_Devices_last_30_days__gc",
        "Last_Login__gc", "Environment_Type__gc",
    ],
    where=[
        {"name": "CompanyId", "operator": "IN", "value": [<all_gsids>], "alias": "A"},
        {"name": "Environment_Type__gc", "operator": "EQ",
         "value": ["1I00D77VYT3R9DKEBCX1PG19GI06464USQPD"], "alias": "B"}  # Standard only
    ],
    where_filter_expression="A AND B",
    limit=1000
)
```

Chunk at 100 GSIDs per call if book > 100 accounts. Merge results.

---

## Tech-touch detection

Before scoring:
```js
var techTouch = !account.CsmName || account.CsmName === 'Jamf Digital Team';
```

Tech-touch accounts use last admin login (not `Last_Timeline_Entry_All__gc`) for R4, and primary product utilization (not worst across all products) for R3. Bundle activation gaps surface as row chips, not score penalties.

---

## Scoring

Load `references/scoring.md` for full R1–R5 formulas, including edge cases and verdict logic.

**Scoring inputs per account (computed from company + relationship data):**
- R1: renewal urgency (renewal date, status)
- R2: account health (Overall_Score__gc, churn flags)
- R3: product adoption (deployment %, login recency, active ratio — tech-touch variant applies)
- R4: activity recency (Last_Timeline_Entry_All__gc — or last admin login for tech-touch)
- R5: expansion signal (Expansion_Rediness_Level__gc, Staircase_Overall_Score__gc)

**Verdict:** act-now / watch / on-track. Auto-renewal accounts downgraded one tier.

---

## Render

Tab 1 of the Radar artifact. Load `references/artifact.md` for full HTML template.

**Swim lanes (accounts with RenewalDate within 0–180 days):**
- Act fast: renewal ≤30 days
- Coming up: 31–60 days
- On horizon: 61–180 days

Within each lane: sort by total score ascending (lowest = most urgent). Then by `Available_to_Renew__gc` descending.

**Heads up strip (above swim lanes):** Accounts outside 180 days that have `Renewal_Status__gc` IN [Escalated, Late Escalated, Notice of Churn] OR `Churn_Risk_Identified__gc=true` OR `Churn_Risk_Level__gc >= 3`. Cap at 5. Sort: escalated first → churn level desc → days to renewal asc.

**Auto-renewal chip:** `Auto_Renewal__gc=true` → "Auto ↑" chip on row. Signals deprioritization in Tab 1 (verdict downgraded); signals expansion opportunity in Tab 2.

**Expand/collapse rows:** Score breakdown + Staircase section (if connected) + Claude synthesis (always).

**Claude synthesis template (per account):**
```
[Account name] [context: renews in X days / renewal status].
[One signal sentence from weakest scoring dimension].
[One action sentence: what to do next].
```

**Staircase section states:**
- MCP not connected: section omitted entirely
- Connected, lookup succeeded: header + narrative
- Connected, lookup failed: header + *"No Staircase match found for [name]."*

---

## Loading state

Render skeleton rows while both queries are pending. See `references/artifact.md` for skeleton HTML. Replace with real rows on data return. On error: replace skeleton with error state — never leave skeleton visible.

---

*Radar capability-renewal.md v2.0 (2026-06-03) — Extracted from Radar SKILL.md v1.3. Full scoring spec in references/scoring.md. Full HTML spec in references/artifact.md.*
