# Sentinel — Capability 7: Top Accounts to Escalate (Leader)
# Aegis v2.0 — June 2026
# https://raw.githubusercontent.com/ToddJamf/aegis/main/sentinel/capability-leader-escalations.md

Load on demand when a Leader-tier user asks what to escalate.

**Triggers:** "what should I escalate" / "top accounts to escalate" / "renewal escalations" / "where should leadership focus".

**Permission:** Leaders have full access — own team by default, region or global on request. No gate.

---

## Scope resolution

| User says | Scope |
|-----------|-------|
| Default / "my team" | `Csm IN [leader.teamGsids]` |
| "[region]" (AMER / EMEIA / APAC / LATAM) | `CS_Territory_Region__gc = <region>` |
| "All" / "global" | No Csm filter |

---

## Sequence (~1-2 calls)

1. **CSM list resolution** — from `leader.teamGsids` or re-query for region/global scope.

2. **Escalation query:**

   ```python
   run_query(
       object_name="company",
       select=["Name", "Csm__gr.Name AS CsmName", "Arr",
               "Renewal_Status__gc", "RenewalDate",
               "Available_to_Renew__gc",
               "Overall_Score__gc",
               "Churn_Risk_Level__gc",
               "Open_CSE_Risk_Mitigation__gc",
               "CS_Territory_Region__gc"],
       where=[
           {"name": "Csm", "operator": "IN", "value": [<scope_csm_gsids>], "alias": "X"},
           {"name": "Account_Status__gc", "operator": "EQ", "value": [<Current_Customer_GSID>], "alias": "Y"},
           {"name": "Renewal_Status__gc", "operator": "IN",
            "value": [<Escalated_GSID>, <Late_Escalated_GSID>], "alias": "A"},
           {"name": "Arr", "operator": "GTE", "value": [50000], "alias": "B"}
       ],
       where_filter_expression="X AND Y AND A AND B",
       sort_by=[{"sortField": "Arr", "sortOrder": "desc"}],
       limit=10
   )
   ```

   Every alias must appear in `where_filter_expression`. Forgetting X or Y anchors silently returns cross-scope results. See `error-handling.md` Section 9.

---

## Ranking + chips

- Default sort: `Arr` DESC
- Compute `days_to_renewal` = today − RenewalDate
- Urgency chip:
  - 🔴 RED: days_to_renewal < 60 OR negative (past due)
  - 🟠 ORANGE: 60 ≤ days_to_renewal ≤ 180
  - 🟢 GREEN: days_to_renewal > 180
- 🚨 if `Renewal_Status__gc = Late Escalated` AND `Open_CSE_Risk_Mitigation__gc = true`

---

## Render

```
🎯 Sentinel — Top Accounts to Escalate
[Leader name]  ·  [scope: team / AMER / global]
[date]

| # | Account | CSM | Region | ARR | Renewal | Days | Status | Health |
|---|---------|-----|--------|-----|---------|------|--------|--------|
| 1 | ...     | ... | ...    | ... | ...     | 🔴.. | ...    | ...    |

[N] accounts qualify · ARR exposed: $[total]  ← only if requester is in manager chain
```

ARR render: standard ARR confidentiality check per row. ARR Band for accounts where requester is outside the account.Csm manager chain.

Output medium: HTML artifact (≥5 rows).

---

## Parametrization

- **ARR floor** — user can override: "ARR floor $X" or "show me everything escalated regardless of ARR"
- **Limit** — default 10; override: "top 25" / "all"
- **Scope** — override at any time: "now show me EMEIA" / "what about global?"
