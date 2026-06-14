# Radar — Identity Reference (CSM / AE persona detection)
# Aegis stack 2026.06.13
# https://raw.githubusercontent.com/ToddJamf/aegis/main/radar/references/identity.md
#
# Radar's LOCAL identity model — resolves the individual-contributor persona (CSM / AE / Hybrid)
# and the book field that scopes every individual capability. This is the dual model:
#   - This file: CSM/AE persona + book_field, for individual capabilities
#     (renewal, expansion, my-plate, my-gaps, update).
#   - shared/identity.md: leader-tier detection (FLL / Segment Leader / SR Leadership / Dept Head),
#     for leader capabilities (compliance, heat-map, portfolio).
# Both load at Step 1 per protocol.md. Don't fold one into the other — they answer different questions.
#
# No-access / unresolved identity → shared/error-handling.md §0 escalate (no dead-end, no polite refusal).

---

## Step 1A — Resolve GSID from email

```python
object = "gsuser"
select = ["Name", "Gsid", "Email", "Employee_Type__gc",
          "CS_Territory_Team__gc", "CS_Territory_Region__gc", "Out_of_Office__gc"]
where  = [
    {"name": "Email",        "operator": "EQ", "value": [<session_email>], "alias": "A"},
    {"name": "IsActiveUser", "operator": "EQ", "value": [True],           "alias": "B"}
]
where_filter_expression = "A AND B"
limit = 1
```

Use the session email. Cache: `identity.gsid`, `identity.name`, `identity.email`. Valid for session.

**No `gsuser` row / permission-denied bounce → not a dead-end, an escalation.** Route to
`shared/error-handling.md` §0 — render the access-escalation block so the user can self-serve a fix
from CS Ops via #help-gainsight. No "couldn't find you" refusal.

---

## Step 1B — Persona detection (parallel counts)

Fire both in a single parallel batch after Step 1A returns. These two COUNT queries decide CSM vs. AE
vs. Hybrid. The AE field is `Primary_Territory_Owner__gc` — keep it; this is the persona-detection axis.

**Query A — CSM book size**
```python
object = "company"
select = ["COUNT(Gsid) AS CsmCount"]
where  = [
    {"name": "Csm",               "operator": "EQ", "value": [<gsid>],                              "alias": "A"},
    {"name": "Account_Status__gc","operator": "EQ", "value": ["1I00D3F36I0KHC861HOKK23F3UVIZTX9YKL7"], "alias": "B"},
    {"name": "True_Customer__gc", "operator": "EQ", "value": [True],                                 "alias": "C"},
    {"name": "Arr",               "operator": "GTE","value": [10000],                                "alias": "D"}
]
where_filter_expression = "A AND B AND C AND D"
```

**Query B — AE book size**
```python
object = "company"
select = ["COUNT(Gsid) AS AeCount"]
where  = [
    {"name": "Primary_Territory_Owner__gc", "operator": "EQ", "value": [<gsid>],                              "alias": "A"},
    {"name": "Account_Status__gc",          "operator": "EQ", "value": ["1I00D3F36I0KHC861HOKK23F3UVIZTX9YKL7"], "alias": "B"},
    {"name": "True_Customer__gc",           "operator": "EQ", "value": [True],                                 "alias": "C"},
    {"name": "Arr",                         "operator": "GTE","value": [10000],                                "alias": "D"}
]
where_filter_expression = "A AND B AND C AND D"
```

Anchor every alias in `where_filter_expression` — it overrides the implicit AND-join
(error-handling §A5). Bulk-call / batch limits per error-handling §A6.

---

## Persona resolution table

| CsmCount | AeCount | Persona | `identity.book_field` | `identity.book_label` |
|---|---|---|---|---|
| > 0 | 0 | CSM | `Csm` | "Your book" |
| 0 | > 0 | AE | `Primary_Territory_Owner__gc` | "Your territory" |
| > 0 | > 0 | Hybrid | `Csm` (default) | "Your book" |
| 0 | 0 | No book | — | → error-handling §0 escalate |

**Hybrid note:** After artifact loads, surface once in chat:
> "Showing your CSM book (N accounts). You also appear as a Territory Owner — say 'AE view' to switch."

**No book → escalate, don't dead-end.** Zero on both counts with a valid `gsuser` row is an
"authenticated, but the seat has no data scope" case → error-handling §0 (`Need: Confirm this user's
book / role assignment in Gainsight`). No flat refusal.

---

## OOO check

If `Out_of_Office__gc = true`: surface once before first data operation:
> "You're marked OOO in Gainsight — pulling your book anyway."

Cache acknowledgment. Don't re-prompt. (Aligned with shared/identity.md OOO + error-handling §B3.)

---

## Cache shape

```python
identity = {
    "name":        <str>,
    "gsid":        <str>,
    "email":       <str>,
    "persona":     "CSM" | "AE" | "Hybrid",
    "book_field":  "Csm" | "Primary_Territory_Owner__gc",
    "book_label":  "Your book" | "Your territory",
    "book_size":   <int>,   # total accounts at $10K+ ARR
    "verified_at": <timestamp>
}
```

For leader capabilities, the leader cache (tier, teamGsids, region, csmToTeam) is built by
shared/identity.md and held alongside this persona cache — they coexist for Hybrid leaders.

---

*2026.06.13 — Ported into the repo from the radar-2026-06-03 archive. Aligned no-record / no-book handling to shared/error-handling.md §0 escalate (removed the two local "hard stop" dead-end refusals). Header → CalVer; framed as the CSM/AE half of the dual identity model alongside shared/identity.md. Added where_filter_expression anchor + error-handling §A5/§A6/§B3 cross-refs. No PII present in source — confirmed clean. AE field Primary_Territory_Owner__gc preserved.*
