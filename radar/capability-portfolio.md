# Radar — Capability: Portfolio / Segment Story
# Aegis stack 2026.06.13
# https://raw.githubusercontent.com/ToddJamf/aegis/main/radar/capability-portfolio.md
#
# FLL / Segment Leader / SR Leadership / Dept Head asking for a portfolio view,
# ARR risk analysis, segment story, or QBR prep.
# Triggers: "portfolio view", "ARR risk", "segment story", "what do I tell my boss",
#           "QBR prep", "renewal exposure", "segment health", "book from 30k feet"
# Permission: FLL and above.
# Data routing (incl. Staircase): shared/source-routing.md. Output/footer: shared/output-discipline.md.
# Report mining (ride admin-built reports over hand-built queries): see § Report mining below.

---

## Purpose

Two-part output:
1. **Numbers (HTML artifact)** — ARR distribution, renewal exposure by window, escalation pool. Data an FLL copies into a QBR slide deck or leadership email.
2. **Narrative (chat)** — 3–5 sentences the FLL can speak from. The story, not the table.

---

## Scope resolution

| User says | Scope |
|-----------|-------|
| Default / "my team" / "my book" | `Csm IN [identity.teamGsids]` |
| "[region]" | `CS_Territory_Region__gc = <region>` |
| "All" / "global" | No Csm filter |

---

## Report mining — ride admin-built reports before hand-building queries

The portfolio numbers (ARR by health, renewal exposure by window, escalation pool) are exactly the
kind of view a Gainsight admin has often already built as a maintained report. Ride it via
`report_search_tool` + `fetch_report_data` over the hand-built aggregations below — admin reports track
the live schema and won't drift the way a hardcoded `run_query` does (and `SUM(CASE WHEN)` is dead in
tenant anyway — P_5026).

- **Reports encode tenant *conventions*** — the canonical health-tier cuts, renewal-window bucketing,
  and escalation definition. Ride these rather than re-deriving them.
- **Report-specific filters encode *scope*** — re-scope to the requester's `identity.teamGsids` or the
  resolved segment, don't inherit the report author's pre-set team/region/ARR floor.

**Fallback:** no suitable report, or scope can't be reconciled → the hand-built queries below stand.

---

## Sequence (~4 calls, parallel)

**Call 1 — ARR by health tier:**
```python
# Do NOT use SUM(CASE WHEN ...) — P_5026.
run_query(
    object_name="company",
    select=[
        "Overall_Score__gc",
        "Renewal_Status__gc",
        "COUNT(Gsid) AS AccountCount",
        "SUM(Arr) AS TotalArr"
    ],
    where=[
        {"name": "Csm", "operator": "IN", "value": [<scope_csm_gsids>], "alias": "A"},
        {"name": "Account_Status__gc", "operator": "EQ",
         "value": ["1I00D3F36I0KHC861HOKK23F3UVIZTX9YKL7"], "alias": "B"}
    ],
    where_filter_expression="A AND B",
    group_by=["Overall_Score__gc", "Renewal_Status__gc"]
)
```

**Call 2 — Renewal exposure by window:**
```python
run_query(
    object_name="company",
    select=["Name", "Arr", "RenewalDate", "Overall_Score__gc",
            "Renewal_Status__gc", "Auto_Renewal__gc",
            "Csm__gr.Name AS CsmName"],
    where=[
        {"name": "Csm", "operator": "IN", "value": [<scope_csm_gsids>], "alias": "A"},
        {"name": "Account_Status__gc", "operator": "EQ",
         "value": ["1I00D3F36I0KHC861HOKK23F3UVIZTX9YKL7"], "alias": "B"},
        {"name": "RenewalDate", "operator": "LTE",
         "value": [<180d_from_today>], "alias": "C"}
    ],
    where_filter_expression="A AND B AND C",
    sort_by=[{"sortField": "RenewalDate", "sortOrder": "asc"}],
    limit=50
)
```

**Call 3 — Escalation pool (ARR at risk):**
```python
run_query(
    object_name="company",
    select=["Name", "Arr", "RenewalDate",
            "Overall_Score__gc", "Renewal_Status__gc",
            "Csm__gr.Name AS CsmName"],
    where=[
        {"name": "Csm", "operator": "IN", "value": [<scope_csm_gsids>], "alias": "A"},
        {"name": "Account_Status__gc", "operator": "EQ",
         "value": ["1I00D3F36I0KHC861HOKK23F3UVIZTX9YKL7"], "alias": "B"},
        {"name": "Renewal_Status__gc", "operator": "IN",
         "value": ["1I00VM2QFXVQWI47QSFAZ1SAZJY69GIITKHG",
                   "1I00S7LFYQ5DJNFTY2ZBXMOD710F5XYR7RWC"],
         "alias": "C"}
    ],
    where_filter_expression="A AND B AND C",
    sort_by=[{"sortField": "Arr", "sortOrder": "desc"}],
    limit=20
)
```

**Call 4 — Staircase (Pattern B — top escalations):**

Same as heat-map Call 4. Up to 5 escalations, high-confidence matches only.

---

## Render

### HTML artifact — numbers

```
Portfolio View — [scope label]  ·  [date]

ARR by Health
  A: $X ([N] accounts) · B: $X · C: $X · D: $X · F: $X

Renewal Exposure (next 180 days)
  ≤30d:   $X ARR · [N] accounts
  31–60d: $X ARR · [N] accounts
  61–90d: $X ARR · [N] accounts
  91–180d: $X ARR · [N] accounts
  Auto-renewal: $X of the above

Escalation Pool
  [N] accounts  ·  $X ARR  ·  [Top 5 listed by ARR]
```

**ARR policy:** Apply `references/arr-policy.md`. List views require ownership check — render `ARR_Band__gc` [restricted] for accounts not owned by the requester's org.

### Chat narrative — segment story

3–5 sentences the FLL can use verbatim:

```
[Portfolio summary] Of [N] accounts totaling $Xk ARR, [N]% are A/B health.
[Renewal exposure] $Xk renews in the next 90 days — [N] of those are at escalation or late status.
[Escalation pool] Biggest risk is [Account] at $Xk — [one sentence on why].
[Win] [Account] renewed cleanly at $Xk — [CSM]'s book is running clean.
[Closing] [One forward-looking sentence — what to watch or where to invest].
```

Tone: executive register. Something the FLL can say in a leadership call, not a data dump.

---

*2026.06.13 — Migrated onto shared layer (header CalVer; data routing → source-routing, output → output-discipline). Added report mining: ride admin-built Gainsight reports via report_search_tool + fetch_report_data over hand-built aggregations (reports encode tenant *conventions*; report-specific filters encode *scope* — re-scope to requester; hand-built queries remain the fallback). P_5026 SUM(CASE WHEN) note preserved.*
*2026-06-03 (v2.0) — Ported from Gainsight Sentinel v3.7 references/portfolio.md.*
