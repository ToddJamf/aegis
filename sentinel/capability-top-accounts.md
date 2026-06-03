# Sentinel — Capability 4: Top Accounts to Work This Week (CSM)
# Aegis v2.0 — June 2026
# https://raw.githubusercontent.com/ToddJamf/aegis/main/sentinel/capability-top-accounts.md

Load on demand when a CSM asks what to work on this week.

**Triggers:** "what's on my plate" / "top accounts this week" / "what should I work on" / "top accounts today" — CSM tier only.

If a Leader asks "top accounts" → do NOT load this file. Route to Cap 7 (Leader escalations) instead.

---

## Scope

Default: own book (`Csm = user.Gsid`).

"Top accounts in my region" → expand to same-region cross-book accounts from identity resolution.

---

## Pagination — use `identity.bookSize`

- **≤100 accounts:** single query, `limit=100`, sort `RenewalDate ASC` / `Arr DESC`.
- **101–200 accounts:** fetch page 1 (`limit=100, offset=0`) + page 2 (`limit=100, offset=100`) in parallel. Merge, then rank.
- **>200 accounts:** two-page fetch (200 total). Add note: *"Your book is [N] accounts — showing the highest-priority 200 sorted by renewal date and ARR."*

`RenewalDate ASC` / `Arr DESC` ensures tactical accounts (imminent renewals) surface in page 1. Only say "your full book ([N] accounts)" when records returned = `identity.bookSize`.

---

## Query

```python
run_query(
    object_name="company",
    select=[<lean overview select per field-registry>],
    where=[
        {"name": "Csm", "operator": "EQ", "value": [<user.Gsid>], "alias": "A"},
        {"name": "Engaged_This_Period_by_CSM__gc", "operator": "EQ", "value": [False], "alias": "B"}
    ],
    where_filter_expression="A AND B",
    sort_by=[
        {"sortField": "RenewalDate", "sortOrder": "asc"},
        {"sortField": "Arr", "sortOrder": "desc"}
    ],
    limit=100
)
```

---

## Ranking — two tiers

### Tactical (act this week)

Escalated status · renewal ≤60 days · sentiment D/F · any combination. Need a move this week. Include one-line "why" per account.

### Strategic (invest this week)

Renewal 60–180 days out · sentiment C or stale (no sentiment + >60d since last touch) · Grow accounts going quiet · high-ARR Retain accounts past cadence. Relationship investments that protect the renewal window.

**Each tier:** top 3–5 accounts. One-line *why*. Total ~7 accounts.

---

## React accounts — email engagement composite

For React accounts: last touch = MAX(`Last_Timeline_Entry_Engagement__gc`, `Last_Timeline_Entry_CSM_Email__gc`). Render:

```
Last call: [date]  ·  Last email: [date]  →  Last touch: [date]
```

Add note: *"React cadence is CTA-driven — email counts as a valid touch for UpMarket KPI. Gainsight compliance flag may not reflect email."*

---

## Output

HTML artifact. Two clearly labeled sections: Tactical and Strategic. Clean columns — account name, category, renewal date, ARR band, sentiment, one-line why.
