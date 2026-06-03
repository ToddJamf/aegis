# Radar — Capability: Team / Segment Heat Map
# Aegis v2.0 — June 2026
# https://raw.githubusercontent.com/ToddJamf/aegis/main/radar/capability-heat-map.md
#
# Load when any CS leader asks about escalations, team risk, wins, or the health balance sheet.
# Triggers: "what should I escalate", "top escalations", "renewal escalations",
#           "team risk", "segment risk", "team heat map", "what's going well",
#           "team wins", "where should leadership focus"
# Permission: FLL, Segment Leader, SR Leadership, Department Head.

---

## Scope resolution

| User says | Scope |
|-----------|-------|
| Default / "my team" / "my segment" / "my patch" | `Csm IN [identity.teamGsids]` |
| "[region]" (AMER / EMEIA / APAC / LATAM) | `CS_Territory_Region__gc = <region>` ⚠ Unverified on company object in Jamf tenant — may return P_5005. Fall back to named-segment scope if filter fails. |
| "[name]'s segment" / "[name]'s team" | Resolve [name] → Gsid → fetch teamGsids → `Csm IN [target.teamGsids]` |
| "All" / "global" | No Csm filter |

**Named-segment resolution:**
1. Extract name from trigger. Resolve via `run_query` on `gsuser` (last-name CONTAINS filter).
2. If multiple matches → disambiguate.
3. Fetch target's downward `teamGsids` (recursive walk, depth 5).
4. Apply `Csm IN [scope.teamGsids]`. Label output: `[scope: [Name]'s segment]`.
5. Authorization: target must be within requester's downward hierarchy OR requester is CS Ops. If not → refuse: *"[Name] isn't in your hierarchy. Scoped views are limited to your own org."*

---

## Sequence (~4 calls, parallel)

**Call 1 — Escalation query:**
```python
run_query(
    object_name="company",
    select=["Name", "Csm__gr.Name AS CsmName", "Csm AS CsmGsid",
            "Renewal_Status__gc", "RenewalDate", "Arr",
            "Available_to_Renew__gc",
            "Overall_Score__gc", "Churn_Risk_Level__gc",
            "Open_CSE_Risk_Mitigation__gc"],
    where=[
        {"name": "Csm", "operator": "IN", "value": [<scope_csm_gsids>], "alias": "X"},
        {"name": "Account_Status__gc", "operator": "EQ",
         "value": ["1I00D3F36I0KHC861HOKK23F3UVIZTX9YKL7"], "alias": "Y"},
        {"name": "Renewal_Status__gc", "operator": "IN",
         "value": ["1I00VM2QFXVQWI47QSFAZ1SAZJY69GIITKHG",   # Escalated
                   "1I00S7LFYQ5DJNFTY2ZBXMOD710F5XYR7RWC"],  # Late Escalated
         "alias": "A"},
        {"name": "Arr", "operator": "GTE", "value": [50000], "alias": "B"}
    ],
    where_filter_expression="X AND Y AND A AND B",
    sort_by=[{"sortField": "Arr", "sortOrder": "desc"}],
    limit=10
)
```

**Call 2 — At-risk (non-escalated):**
```python
run_query(
    object_name="company",
    select=["Name", "Csm__gr.Name AS CsmName", "Arr", "RenewalDate",
            "Overall_Score__gc", "CSM_Sentiment__gc",
            "Churn_Risk_Level__gc", "Churn_Risk_Identified__gc"],
    where=[
        {"name": "Csm", "operator": "IN", "value": [<scope_csm_gsids>], "alias": "A"},
        {"name": "Account_Status__gc", "operator": "EQ",
         "value": ["1I00D3F36I0KHC861HOKK23F3UVIZTX9YKL7"], "alias": "B"},
        {"name": "Overall_Score__gc", "operator": "IN",
         "value": ["D", "F"], "alias": "C"},
        {"name": "Renewal_Status__gc", "operator": "NOT_IN",
         "value": ["1I00VM2QFXVQWI47QSFAZ1SAZJY69GIITKHG",
                   "1I00S7LFYQ5DJNFTY2ZBXMOD710F5XYR7RWC"],
         "alias": "D"}
    ],
    where_filter_expression="A AND B AND C AND D",
    sort_by=[{"sortField": "Arr", "sortOrder": "desc"}],
    limit=10
)
```

**Call 3 — Wins (positive sentiment, high ARR):**
```python
run_query(
    object_name="company",
    select=["Name", "Csm__gr.Name AS CsmName", "Arr",
            "RenewalDate", "Renewal_Status__gc",
            "CSM_Sentiment__gc", "Overall_Score__gc"],
    where=[
        {"name": "Csm", "operator": "IN", "value": [<scope_csm_gsids>], "alias": "A"},
        {"name": "Account_Status__gc", "operator": "EQ",
         "value": ["1I00D3F36I0KHC861HOKK23F3UVIZTX9YKL7"], "alias": "B"},
        {"name": "CSM_Sentiment__gc", "operator": "IN",
         "value": ["1I005ZWPK9ZBV2E3EMWRPESWDAWVK4R60KIZ",   # A
                   "1I005ZWPK9ZBV2E3EMWIB8BE7T1KHE98E7X1"],  # B
         "alias": "C"}
    ],
    where_filter_expression="A AND B AND C",
    sort_by=[{"sortField": "Arr", "sortOrder": "desc"}],
    limit=5
)
```

**Call 4 — Staircase (Pattern B — top escalations):**
```python
staircase_targets = (escalations or [])[:5]
lookup_results = [staircase_account_lookup(name=acct["Name"]) for acct in staircase_targets]
staircase_signals = [
    staircase_analyze_account(
        account_id=match[0]["account_id"],
        query="Current risk and concerns for this account?"
    )
    for match in lookup_results
    if match and match[0].get("confidence") == "high"
]
```

If Staircase not connected → skip, label ⚠ unavailable.

**csmToTeam attribution (Segment Leader+ only):**
If `identity.csmToTeam` is populated (built during org walk for csm_depth ≥ 2), render a Team column in escalation and at-risk tables showing the FLL attribution for each CSM. Gives Segment Leaders visibility into which sub-team owns the risk.

---

## Render

HTML artifact.

```
🗺 Team Heat Map — [scope label]  ·  [date]
[N] accounts  ·  $X total ARR  ·  [N] escalations  ·  [N] at risk

🚨 Escalations ([N])
  [Account]  ·  $ARR  ·  CSM: [Name]  ·  [Team: name]  ·  [status chip]  ·  Renewal: [date]
  [🚨 CSE mitigation active]
  Staircase: [1–2 sentence signal]  ← if available
  ...

⚠️ At Risk — not escalated ([N])
  [Account]  ·  $ARR  ·  CSM: [Name]  ·  [Team: name]  ·  Health: [score]  ·  Sentiment: [grade]
  ...

🏆 Wins this period ([N])
  [Account]  ·  $ARR  ·  CSM: [Name]  ·  Sentiment: [A/B]  ·  [renewal date if clean]
  ...

💬 Leadership talking points
  [3 synthesized bullets — what to tell a VP, what needs executive attention, what's going well]
```

**Leadership talking points — synthesis rules:**
1. Top escalation by ARR — *"[Account] at $Xk is the biggest open escalation — CSE mitigation is active. Needs exec visibility if not already flagged."*
2. At-risk pattern — *"3 D/F accounts totaling $Xk in AMER — all pending renewal Q3. Not yet escalated but trending the wrong way."*
3. Win worth naming — *"[Account] at $Xk renewed cleanly, sentiment A — strong signal from [CSM]'s team."*

**Omit sections if empty** — do not render empty escalation or wins sections.

---

*Radar capability-heat-map.md v2.0 (2026-06-03) — Ported from Gainsight Sentinel v3.7 references/heat-map.md. P_5005 region filter caveat preserved.*
